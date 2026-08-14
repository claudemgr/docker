# COMPOSEMGR.md — Master Spec for composemgr Compose Repos

This file is the master template for every repository in the `composemgr` GitHub
organization. It is copied verbatim into each repo as `AI.md` and is permanent —
never delete it, never rewrite its structure, only sync it from the master copy in
`claudemgr/docker/COMPOSEMGR.md`.

Placeholders: `{name}` = repository/stack name (e.g. `vaultwarden`) · `{service}` =
service key inside the stack (e.g. `app`, `db`) · `{port}` = the service's internal
container port · `{ext_port}` = the host-side published port · `{image}` = the
upstream (or `casjaysdevdocker/*`) image reference.

# PART 0 — Repository Identity

- **GitHub org:** `composemgr` — one repository per compose stack, ~300 repos
- **Purpose:** ready-to-run `docker compose` stacks for self-hosted applications
- **Managed by:** the `composemgr` tool (`composemgr install {name}`); repos are also
  usable standalone with plain `docker compose up -d`
- **No generator tooling** — unlike `dockersrc`/`casjaysdevdocker` repos there is NO
  `gen-dockerfile`/`gen-script` involvement; every file is hand-maintained
- **No CI/CD** — no `.github/workflows/`, no `.gitea/workflows/`, no build pipelines;
  never create any workflow file in these repos
- **Images used:** upstream official images, or `casjaysdevdocker/{name}` images when
  a CasjaysDev app image exists for the application

## The composemgr tool

`composemgr` (see `composemgr --help`) is the deployment layer around these repos:
`install` fetches a stack by name/URL, `up`/`down`/`restart`/`logs`/`ps` wrap compose,
`backup` snapshots project data, `network` manages the shared docker networks,
`nginx {host} {url}` generates the reverse-proxy config from the line-1 proxy comment,
and `repo pull|push|commit` handles git ops.

Its env pipeline (`composemgr env` / `__update_env_file`) generates the runtime env
file: it sources the repo's `app.env`, `~/.config/secure/cloudflare.txt`, detects
`HOST_IP_4`/`HOST_IP_6` from the default-route interface, and on first run derives
`BASE_HOST_NAME` as `{name}.$BASE_DOMAIN_NAME`. The variables it exports are the
vocabulary in PART 3 — compose files consume them, never define their own names.

**Env precedence:** the composemgr-generated default env file is the preferred and
authoritative source of values — the full variable set lives in the composemgr
script itself (`__update_env_file`), which is the canonical list. The repo's
`app.env` exists only to overwrite the few selected vars a deployment needs to
change; it must never restate the whole set.

**Why every reference is `${VAR:-fallback}`:** composemgr exports the real values when
it runs the stack; the hardcoded safe fallbacks exist so the same compose file also
works WITHOUT composemgr — plain `docker compose up -d`, zero config. Never remove a
fallback, and never make a stack depend on composemgr being installed.

The compose template embedded in the script (`composemgr new`) is a starting
skeleton only — repos are aligned to THIS spec, not to the script's inline template;
where they differ (labels, ports), this spec wins.

# PART 1 — Non-Negotiable Rules

1. **One stack per repo** — a single `docker-compose.yaml` defining the whole stack
2. **Labels are optional and OFF by default** — new/regenerated compose files carry NO
   `labels:` block; label blocks are added only on explicit request ("add cloudflare",
   "add traefik") using the exact templates in PART 4
3. **Networks default to the minimal block** — one stack-private network named after the
   repo, `external: false` (PART 3); the `proxy`/`cloudflare` external networks are added
   only together with the matching label block
4. **Ports bind to `172.17.0.1`** — never `0.0.0.0`; the stack is fronted by a reverse
   proxy on the docker host (the nginx proxy address comment on line 1 records it)
5. **Published ports are random from `63000`–`64999`** — when creating a stack (or
   assigning a new port), pick a random port in that range that no other composemgr
   repo already uses; verify with
   `grep -rh -- '172.17.0.1:' ../*/docker-compose.yaml | grep -o ':[0-9]*:' | sort -u`
   before assigning. The line-1 proxy comment and README Access section must match.
   Existing repos keep their current ports — never renumber a deployed stack
6. **Hardcoded fallbacks are mandatory** — every env reference is `${VAR:-fallback}`
   so the stack runs without composemgr (PART 0); never remove a fallback
7. **No secrets in the repo** — every credential is `${VAR:-changeme_*}`; real values
   live in `app.env` (gitignored) and are exported by composemgr at run time
8. **`volumes/` is runtime state** — bind mounts live under `./volumes/`; the directory
   is never committed (gitignored), only referenced
9. **No CI/CD files, no Dockerfiles** — these repos deploy images, they never build them
10. **Standard env var vocabulary** (PART 3) — only use names composemgr exports; never
    invent new names for concepts that already have one
11. **Comments above, never inline** — and none inside pure data values
12. **Commit via `gitcommit --dir {abs_path} all` only** — never `git commit`/`git push`

# PART 2 — Standard Repository Tree

```
{name}/
├── docker-compose.yaml     # the stack — the only functional file
├── app.env.sample          # documented env template → copy to app.env
├── .gitattributes
├── .gitignore              # ignores app.env, volumes/
├── LICENSE.md              # MIT
├── README.md               # PART 6 layout
└── AI.md                   # this spec (copied from claudemgr/docker/COMPOSEMGR.md)
```

Minimal repos (no configurable env) may omit `app.env.sample`. Nothing else belongs at
the root. `IDEA.md`, `CLAUDE.md`, `TODO.AI.md` are allowed project files when needed.

# PART 3 — docker-compose.yaml Canon

## File anatomy, in order

```yaml
# nginx proxy address - http://172.17.0.1:{ext_port}

name: {name}
x-logging: &default-logging
  driver: json-file
  options:
    max-size: '5m'
    max-file: '1'

services:
  {service}:
    image: {image}:latest
    container_name: {name}-{service}
    hostname: ${BASE_HOST_NAME:-$HOSTNAME}
    restart: always
    pull_policy: always
    logging: *default-logging
    environment:
      TZ: ${TZ:-America/New_York}
      CONTAINER_NAME: {name}-{service}
      HOSTNAME: ${BASE_HOST_NAME:-$HOSTNAME}
      # app-specific vars follow, using the standard vocabulary below
    ports:
      - '172.17.0.1:{ext_port}:{port}'
    volumes:
      - './volumes/data/{name}:/data'
    networks:
      - {name}

networks:
  {name}:
    name: {name}
    external: false
```

## Rules per section

- **Line 1** is always the proxy comment: `# nginx proxy address - http://172.17.0.1:{ext_port}`
  pointing at the primary web port of the stack
- **`name:`** equals the repo name
- **`x-logging`** anchor is defined once and applied to every service via
  `logging: *default-logging`
- **Every service** sets `container_name: {name}-{service}`, `restart: always`,
  `pull_policy: always`, `logging: *default-logging`, and `TZ` + `CONTAINER_NAME` env vars
- **Primary service key** is `app`; databases are `db` (container `{name}-db`); additional
  services take short descriptive keys
- **Volumes** are relative bind mounts: app data `./volumes/data/{name}`, database data
  `./volumes/data/db/{engine}/{name}` (e.g. `./volumes/data/db/postgres/gitea`)
- **Databases** get a `healthcheck` and the app depends on them with
  `condition: service_healthy`; db services join only the stack-private network and
  publish no ports
- **Multi-port stacks** publish each additional port on `172.17.0.1` too; non-HTTP
  listener exceptions (e.g. SSH `'2222:22'`) are allowed when the service genuinely
  needs LAN exposure

## Standard env var vocabulary

The authoritative, always-current list is `__update_env_file` in the composemgr
script — when in doubt, read the script, not this table. The tables below are a
reference snapshot of the variables composemgr exports. Compose files may only
reference names from this vocabulary; app-native variable names
(`GITEA__server__DOMAIN`, `ADMIN_TOKEN`, …) are set from these via
`APP_NATIVE_VAR: ${STANDARD_VAR:-fallback}` in the `environment:` block.

### Core

| Variable | Meaning | Default pattern |
|----------|---------|-----------------|
| `TZ` | timezone | `America/New_York` |
| `BASE_HOST_NAME` | FQDN the app is served as | `$HOSTNAME` |
| `BASE_DOMAIN_NAME` | bare domain | `example.com` |
| `HOST_IP_4` / `HOST_IP_6` | docker host's LAN IPs (detected) | empty |
| `NGINX_PROXY_URL` | proxy URL from the line-1 comment | empty |
| `BASE_DIR_STORAGE` | host storage root | empty |
| `BASE_DIR_DATABASES` | host database root | empty |
| `SERVICE_USER` / `SERVICE_GROUP` | run-as user/group | empty |
| `CLOUDFLARE_ZONE_NAME` | zone for cloudflare labels | empty |

### App identity, users & secrets

| Variable | Meaning | Default pattern |
|----------|---------|-----------------|
| `APP_ORG_NAME` | org/display name in UI and mails | `{name}` |
| `APP_RUN_AS` | in-container run-as user | empty |
| `APP_ADMIN_USER` / `APP_ADMIN_PASS` | initial admin account | `changeme_*` |
| `APP_USER_NAME` / `APP_USER_PASS` | initial regular account | `changeme_*` |
| `APP_SECRET_KEY` | primary secret/admin token | `changeme_secret_key_min_32_chars` |
| `APP_INTERNAL_TOKEN` | secondary internal token | `changeme_internal_token_min_32_chars` |
| `APP_SECRET_TOKEN_16/32/64` | fixed-length secret tokens | `changeme_*` |
| `APP_JWT_TOKEN` / `APP_API_TOKEN` | JWT / API tokens | `changeme_*` |
| `APP_TEMP_PASS` | temporary first-run password | `changeme_*` |
| `RPC_SECRET` / `ENCRYPTION_KEY` | app-specific secrets | `changeme_*` |
| `BACKUPS_PW` | backup encryption password | `changeme_*` |

### Databases

| Variable | Meaning | Default pattern |
|----------|---------|-----------------|
| `DB_CREATE_DATABASE_NAME` | database name | `{name}` |
| `DB_USER_NAME` / `DB_USER_PASS` | database user/password | `{name}` / `changeme_db_password` |
| `DB_ADMIN_NAME` / `DB_ADMIN_PASS` | database superuser | `changeme_admin_password` |
| `POSTGRESQL_URL`, `MARIADB_URL`, `REDIS_URL`, `VALKEY_URL`, `MONGODB_URL`, `COUCHDB_URL`, `MSSQLDB_URL`, `SUPABASE_URL`, `COUCHBASE_URL`, `POCKETBASE_URL` | external shared-DB connection URLs | empty |

### Email

| Variable | Meaning | Default pattern |
|----------|---------|-----------------|
| `EMAIL_SERVER_HOST` | SMTP host | `172.17.0.1` |
| `EMAIL_SERVER_PORT` | SMTP port | `587` |
| `EMAIL_SERVER_LOGIN_NAME` / `EMAIL_SERVER_LOGIN_PASS` | SMTP credentials | empty |
| `EMAIL_SERVER_MAIL_FROM` | From address | `noreply@example.com` |
| `EMAIL_SERVER_FROM_ORG` | From display name | `$APP_ORG_NAME` |

Every reference uses the `${VAR:-default}` form so the stack works with zero config
(rule 6).

## Getting values into containers

Two mechanisms, in order of preference:

1. **Inline mapping (preferred)** — wherever the image accepts a specific env var,
   map the standard vocabulary onto it in the `environment:` block:

   ```yaml
       environment:
         ADMIN_TOKEN: ${APP_SECRET_KEY:-changeme_secret_key_min_32_chars}
         GITEA__database__PASSWD: ${DB_USER_PASS:-changeme_db_password}
   ```

2. **env_file loading (fallback)** — when the image reads its own native variable
   names directly (or there are too many to sensibly map one-by-one), set the
   image-specific vars in `app.env` and hand the env files straight to the
   container via an anchor (real pattern from the icecast repo):

   ```yaml
   x-env-file: &env_file
     - .env
     - app.env
   ```

   applied per service with `env_file: *env_file`. `.env` is the
   composemgr-generated defaults file; `app.env` carries the deployment's
   overrides.

Both mechanisms may coexist on one service — compose gives `environment:` entries
precedence over `env_file`, so mapped standard vars stay authoritative.

# PART 4 — Optional Label Blocks (add on request only)

By default a compose file has **no `labels:` block** and only the minimal network set.
When the user asks to "add cloudflare" or "add traefik" (or both), add the exact block
below to the requested service and extend `networks` accordingly. Never add these
unprompted, and never invent label variants.

## "add cloudflare"

Service labels:

```yaml
    labels:
      - 'cloudflare.enable=true'
      - 'cloudflare.service=http://{name}-{service}:{port}'
      - 'cloudflare.hostname={name}.${CLOUDFLARE_ZONE_NAME:-}'
```

Service networks gain `cloudflare`; top-level `networks:` gains:

```yaml
  cloudflare:
    external: true
```

## "add traefik"

Service labels:

```yaml
    labels:
      - 'traefik.enable=true'
      - 'traefik.docker.network=proxy'
      - 'traefik.http.routers.{name}-{service}.entrypoints=http'
      - 'traefik.http.routers.{name}-{service}-secure.tls=true'
      - 'traefik.http.routers.{name}-{service}.rule=Host(`${BASE_HOST_NAME:-$HOSTNAME}`)'
      - 'traefik.http.middlewares.{name}-{service}-https-redirect.redirectscheme.scheme=https'
      - 'traefik.http.routers.{name}-{service}.middlewares={name}-{service}-https-redirect'
      - 'traefik.http.routers.{name}-{service}-secure.entrypoints=https'
      - 'traefik.http.routers.{name}-{service}-secure.rule=Host(`${BASE_HOST_NAME:-$HOSTNAME}`)'
      - 'traefik.http.routers.{name}-{service}-secure.tls.certresolver=cloudflare'
      - 'traefik.http.routers.{name}-{service}-secure.service={name}-{service}'
      - 'traefik.http.services.{name}-{service}.loadbalancer.server.port={port}'
```

Service networks gain `proxy`; top-level `networks:` gains:

```yaml
  proxy:
    external: true
```

## Combined ("add cloudflare and traefik")

Merge both label lists under one `labels:` block (traefik+cloudflare enable lines first,
matching existing repos), add both external networks. `{port}` in every label is the
service's **internal** container port, never the published one.

**Host rule syntax:** traefik matcher values take backticks —
``Host(`${BASE_HOST_NAME:-$HOSTNAME}`)``. Existing repos carry a legacy
`Host($(...))` form; when touching a repo that has labels, correct its Host rules
to the backtick form as part of the alignment.

# PART 5 — app.env.sample Canon

`app.env.sample` is the documented template for the deployment's override file:
the user copies it to `app.env` and changes the values for their setup. That
`app.env` then feeds the stack two ways — composemgr sources it when generating
the runtime env, and services that use the env_file mechanism (PART 3) load it
directly.

```sh
# {Name} environment configuration
# Copy to app.env and customize before starting

# Timezone
TZ=America/New_York

# Hostname
BASE_HOST_NAME={name}.example.com
```

- Header comment names the app and states the copy-to-`app.env` instruction
- Grouped by concern with a comment above each group (Timezone, Hostname, Secrets,
  Database, Email / SMTP, …)
- **Selected overrides only** — `app.env` overwrites the few vars a deployment
  actually needs to change; the full set comes from the composemgr-generated
  default env file (PART 0 precedence). Never dump the whole vocabulary into the
  sample
- Only variables the compose file actually references — keep it in sync; a var in the
  sample that no service consumes is a bug
- Values are the same safe defaults the compose file falls back to

# PART 6 — README.md Canon

Section order (emoji headers, matching existing repos):

1. `## 👋 Welcome to {name} 🚀` — one-line upstream description
2. `## 📋 Description`
3. `## 🚀 Services` — bullet list `- **{service}**: {image}:{tag}`
4. `## 📦 Installation` — three options, in this order:
   - Quick Install: `curl -q -LSsf "https://raw.githubusercontent.com/composemgr/{name}/main/docker-compose.yaml" -o compose.yml`
   - Git Clone: `git clone "https://github.com/composemgr/{name}" ~/.local/srv/docker/{name}` + `docker compose up -d`
   - Using composemgr: `composemgr install {name}`
5. `## 🔧 Configuration` — key env vars in a `shell` fence + pointer to the compose file
6. `## 🌐 Access` — `- **Web Interface**: http://172.17.0.1:{ext_port}`
7. `## 📂 Volumes` — each bind mount with a short purpose
8. `## 🔐 Security` — standard hardening bullets
9. `## 🔍 Logging` — `docker compose logs -f {service}`
10. `## 🛠️ Management` — start/stop/pull-update/logs/restart command block
11. `## 📋 Requirements` — Docker Engine 20.10+, Compose V2+
12. `## 🤝 Author` — casjay + composemgr footer links

Facts (ports, services, volumes, env vars) must match `docker-compose.yaml` exactly —
the compose file is the source of truth. Repos with deliberately richer hand-crafted
READMEs keep their structure; update facts only.

# PART 7 — Verification & Commit

Before every commit:

1. `docker compose -f docker-compose.yaml config -q` — must exit 0 (run with defaults,
   no `app.env`, proving the zero-config guarantee)
2. README facts diffed against the compose file (ports, images, volumes, env vars)
3. `app.env.sample` vars all consumed by the compose file, and vice versa for any
   var without a safe inline default
4. If labels were requested: label `{port}` values equal the container port, Host
   rules use the backtick form, and the matching external networks exist at both
   service and top level
5. Any newly assigned published port is inside `63000`–`64999` and collides with no
   other composemgr repo (rule 5); line-1 comment and README Access match it
6. Write `.git/COMMIT_MESS`, re-read it against the diff, then
   `gitcommit --dir {abs_path} all`

# PART 8 — Examples (from real repos)

Ports shown are the repos' existing (pre-range) assignments — existing stacks keep
their ports (rule 5); a NEW stack would draw a random port from `63000`–`64999`.

## Single-service stack (vaultwarden, default form — no labels)

```yaml
# nginx proxy address - http://172.17.0.1:59090

name: vaultwarden
x-logging: &default-logging
  driver: json-file
  options:
    max-size: '5m'
    max-file: '1'

services:
  app:
    image: vaultwarden/server:latest
    container_name: vaultwarden-app
    hostname: ${BASE_HOST_NAME:-$HOSTNAME}
    restart: always
    pull_policy: always
    logging: *default-logging
    environment:
      TZ: ${TZ:-America/New_York}
      CONTAINER_NAME: vaultwarden-app
      HOSTNAME: ${BASE_HOST_NAME:-$HOSTNAME}
      DOMAIN: https://${BASE_HOST_NAME:-$HOSTNAME}
      ADMIN_TOKEN: ${APP_SECRET_KEY:-changeme_secret_key_min_32_chars}
      SMTP_HOST: ${EMAIL_SERVER_HOST:-172.17.0.1}
      SMTP_PORT: ${EMAIL_SERVER_PORT:-587}
      SMTP_FROM: ${EMAIL_SERVER_MAIL_FROM:-no-reply@${BASE_HOST_NAME:-$HOSTNAME}}
      SMTP_FROM_NAME: ${APP_ORG_NAME:-vaultwarden}
      SIGNUPS_ALLOWED: true
      INVITATIONS_ALLOWED: true
    ports:
      - '172.17.0.1:59090:80'
    volumes:
      - './volumes/data/vaultwarden:/data'
    networks:
      - vaultwarden

networks:
  vaultwarden:
    name: vaultwarden
    external: false
```

## App + database stack (gitea, default form — no labels)

Shows: db service naming, healthcheck + `depends_on`, per-engine volume path, db on the
private network only, double-underscore config injection, extra non-HTTP port.

```yaml
# nginx proxy address - http://172.17.0.1:3001

name: gitea
x-logging: &default-logging
  driver: json-file
  options:
    max-size: '5m'
    max-file: '1'

services:
  app:
    image: gitea/gitea:latest
    container_name: gitea-app
    hostname: ${BASE_HOST_NAME:-$HOSTNAME}
    restart: always
    pull_policy: always
    logging: *default-logging
    environment:
      TZ: ${TZ:-America/New_York}
      CONTAINER_NAME: gitea-app
      HOSTNAME: ${BASE_HOST_NAME:-$HOSTNAME}
      GITEA__DEFAULT__APP_NAME: ${APP_ORG_NAME:-Gitea}
      GITEA__server__DOMAIN: ${BASE_HOST_NAME:-$HOSTNAME}
      GITEA__server__ROOT_URL: http://${BASE_HOST_NAME:-$HOSTNAME}:3001
      GITEA__server__HTTP_PORT: 3000
      GITEA__server__SSH_PORT: 2222
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: gitea-db:5432
      GITEA__database__NAME: ${DB_CREATE_DATABASE_NAME:-gitea}
      GITEA__database__USER: ${DB_USER_NAME:-gitea}
      GITEA__database__PASSWD: ${DB_USER_PASS:-changeme_db_password}
      GITEA__security__SECRET_KEY: ${APP_SECRET_KEY:-changeme_secret_key_min_32_chars}
    ports:
      - '172.17.0.1:3001:3000'
      - '2222:22'
    volumes:
      - './volumes/data/gitea:/data'
      - '/etc/timezone:/etc/timezone:ro'
      - '/etc/localtime:/etc/localtime:ro'
    depends_on:
      gitea-db:
        condition: service_healthy
    networks:
      - gitea

  gitea-db:
    image: postgres:latest
    container_name: gitea-db
    restart: always
    pull_policy: always
    logging: *default-logging
    environment:
      TZ: ${TZ:-America/New_York}
      CONTAINER_NAME: gitea-db
      POSTGRES_DB: ${DB_CREATE_DATABASE_NAME:-gitea}
      POSTGRES_USER: ${DB_USER_NAME:-gitea}
      POSTGRES_PASSWORD: ${DB_USER_PASS:-changeme_db_password}
    volumes:
      - './volumes/data/db/postgres/gitea:/var/lib/postgresql/data'
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${DB_USER_NAME:-gitea} -d ${DB_CREATE_DATABASE_NAME:-gitea}']
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - gitea

networks:
  gitea:
    name: gitea
    external: false
```

## Label add-on applied ("add cloudflare and traefik" on vaultwarden `app`)

Only the delta against the default single-service example — the `app` service gains:

```yaml
    networks:
      - vaultwarden
      - proxy
      - cloudflare
    labels:
      - 'traefik.enable=true'
      - 'cloudflare.enable=true'
      - 'traefik.docker.network=proxy'
      - 'cloudflare.service=http://vaultwarden-app:80'
      - 'cloudflare.hostname=vaultwarden.${CLOUDFLARE_ZONE_NAME:-}'
      - 'traefik.http.routers.vaultwarden-app.entrypoints=http'
      - 'traefik.http.routers.vaultwarden-app-secure.tls=true'
      - 'traefik.http.routers.vaultwarden-app.rule=Host(`${BASE_HOST_NAME:-$HOSTNAME}`)'
      - 'traefik.http.middlewares.vaultwarden-app-https-redirect.redirectscheme.scheme=https'
      - 'traefik.http.routers.vaultwarden-app.middlewares=vaultwarden-app-https-redirect'
      - 'traefik.http.routers.vaultwarden-app-secure.entrypoints=https'
      - 'traefik.http.routers.vaultwarden-app-secure.rule=Host(`${BASE_HOST_NAME:-$HOSTNAME}`)'
      - 'traefik.http.routers.vaultwarden-app-secure.tls.certresolver=cloudflare'
      - 'traefik.http.routers.vaultwarden-app-secure.service=vaultwarden-app'
      - 'traefik.http.services.vaultwarden-app.loadbalancer.server.port=80'
```

And top-level `networks:` becomes:

```yaml
networks:
  vaultwarden:
    name: vaultwarden
    external: false
  proxy:
    external: true
  cloudflare:
    external: true
```
