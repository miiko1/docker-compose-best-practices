# Docker Compose stack: best practices and reusable template

## Scope

This template targets a current Docker Compose v2 release on Linux, usually on a single host. Some fields, including `develop.watch`, are unavailable in early v2 releases. The examples use an application plus PostgreSQL; replace the image names, ports, health check, watch paths, and resource limits with values appropriate for your application.

A Compose file alone does not provide production readiness. You still need TLS and firewall configuration, host patching, image scanning, monitoring, secret rotation, tested backups, and a rollback procedure. Swarm and Kubernetes also have different deployment and secret semantics.

## Best-practice checklist

1. **Make deployments reproducible.** Omit the obsolete top-level `version:` field. Give the project a name, avoid fixed `container_name` values, and use reviewed image tags or immutable digests rather than `latest`. Treat digest updates as reviewed code changes.
2. **Keep the base file production-safe.** Put local builds, source synchronization, debug settings, and other developer conveniences in `compose.dev.yaml`. Put immutable images, restart policy, and measured resource limits in `compose.production.yaml`. Use `include` for modular ownership, not as a substitute for environment overrides.
3. **Separate configuration from secrets.** Use `.env` files for Compose interpolation and non-secret settings. Explicitly pass container environment variables with `environment` or `env_file`; a root `.env` file is not automatically injected into every container. Grant secrets only to services that need them. Plain Compose bind-mounts `file:` secrets from the host; it does not turn them into an externally managed, encrypted secret store.
4. **Use least privilege.** Build small, multi-stage images and run as a non-root user. Where supported by the application, use a read-only root filesystem, a small writable `tmpfs`, `no-new-privileges`, and dropped Linux capabilities. Do not enable `privileged`, host networking, or broad host bind mounts without a documented need.
5. **Expose only what is necessary.** Use service names for internal DNS. Keep databases off host ports and place them on an internal network. Bind application ports to localhost when a same-host reverse proxy provides external access.
6. **Model readiness and failure.** Add meaningful health checks and use `depends_on` with `service_healthy`. Add restart policies for long-running services, but remember that an unhealthy status alone does not make plain Compose restart a container. Applications must still use timeouts, retries with backoff, and reconnection logic.
7. **Preserve and protect data.** Use named volumes for persistent state. Keep volume lifecycle separate from application deployment, test backup and restore procedures, and never use `docker compose down -v` as a routine cleanup command.
8. **Bound resource use and logs.** Set CPU, memory, and PID limits based on measurements. Configure log rotation or forwarding, emit structured application logs to stdout/stderr, and monitor health, disk usage, restarts, and resource pressure.
9. **Handle lifecycle deliberately.** Support `SIGTERM`, allow a sensible stop grace period, run migrations as controlled one-shot operations, and design safe deployment and rollback steps.
10. **Validate the resolved model.** Run `docker compose config` in CI and before deployment to inspect merged files and interpolation. Review remote `include` or `extends` content before executing it; pin remote references to digests where possible.

## Suggested project layout

```text
.
├── compose.yaml                 # Shared, production-safe baseline
├── compose.dev.yaml             # Local build, watch, and debug settings
├── compose.production.yaml      # Immutable images and production limits
├── .env.example                 # Safe example defaults; commit this
├── .env                         # Local non-secret settings; do not commit
├── .env.production              # Deployment settings; protect and do not commit
├── .dockerignore
├── .gitignore
├── Dockerfile
└── secrets/
    └── db_password.txt          # Local development only; do not commit
```

For a larger production deployment, replace file-backed development secrets with your platform's secret manager or externally managed secrets. For `file:` secrets, Docker Compose ignores service-level `uid`, `gid`, and `mode` settings because it uses a bind mount; set suitably restrictive ownership and permissions on the source file and verify that every intended container user can read it.

## `compose.yaml`

```yaml
# Compose v2: do not add the obsolete top-level "version:" field.
name: myapp

x-default-logging: &default-logging
  driver: local
  options:
    max-size: "10m"
    max-file: "3"

services:
  app:
    # Development fallback; production requires an approved immutable reference.
    image: ${APP_IMAGE:-myapp:local}
    restart: unless-stopped
    init: true

    # Ensure the image supports this UID/GID and that it owns required writable
    # paths. Create a matching user if the application needs named-user lookups.
    user: "10001:10001"
    read_only: true
    tmpfs:
      - /tmp:size=64m,mode=1777
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

    # Defaults to local-only access. Use a same-host reverse proxy for public TLS.
    ports:
      - "${APP_BIND:-127.0.0.1}:${APP_PORT:-8080}:8080"

    environment:
      APP_ENV: ${APP_ENV:-development}
      LOG_LEVEL: ${LOG_LEVEL:-info}
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: ${POSTGRES_DB:-app}
      DB_USER: ${POSTGRES_USER:-app}
      # Your application must implement reading this file.
      DB_PASSWORD_FILE: /run/secrets/db_password

    secrets:
      - db_password

    depends_on:
      db:
        condition: service_healthy
        restart: true

    # Replace with an executable included in your image. It must test real
    # readiness, exit 0 on success and non-zero on failure, and finish quickly.
    healthcheck:
      test: ["CMD", "/app/healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s

    stop_grace_period: 30s
    networks:
      - edge
      - data
    logging: *default-logging

  db:
    # Use a major tag for local development only; pin an exact version/digest in prod.
    image: ${POSTGRES_IMAGE:-postgres:18}
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-app}
      POSTGRES_USER: ${POSTGRES_USER:-app}
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - type: volume
        source: db_data
        # PostgreSQL 18+ uses a major-version-specific PGDATA below this mount.
        # For PostgreSQL 17 and earlier, target /var/lib/postgresql/data instead.
        target: /var/lib/postgresql
    healthcheck:
      # $$ passes $VAR through Compose so PostgreSQL evaluates it at runtime.
      test: ["CMD-SHELL", "pg_isready -U \"$${POSTGRES_USER}\" -d \"$${POSTGRES_DB}\""]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - data
    # Intentionally no host "ports:" entry for PostgreSQL.
    logging: *default-logging

volumes:
  db_data:

networks:
  edge:
  data:
    internal: true

secrets:
  db_password:
    file: ${DB_PASSWORD_FILE:-./secrets/db_password.txt}
```

The volume target follows the [PostgreSQL image's PostgreSQL 18+ layout](https://hub.docker.com/_/postgres#pgdata): `PGDATA` is `/var/lib/postgresql/18/docker` for PostgreSQL 18, while the volume is mounted at `/var/lib/postgresql`. PostgreSQL 17 and earlier use `/var/lib/postgresql/data` instead. Do not point a different PostgreSQL major version at an existing data directory and treat it as an in-place upgrade; back up the database and follow the image's `pg_upgrade` guidance.

The [official image](https://hub.docker.com/_/postgres#postgres_user) creates `POSTGRES_USER` as a **superuser**. This compact example reuses that initialization role for the application so it can run without a separate provisioning script. Before production use, provision a separate least-privileged application role with its own secret, and grant only the privileges it needs; use separate migration and runtime roles where practical.

## `compose.dev.yaml`

```yaml
services:
  app:
    build:
      context: .
      target: development
    image: myapp:local
    restart: "no"
    read_only: false
    environment:
      APP_ENV: development
      LOG_LEVEL: debug
    develop:
      watch:
        # Adapt these paths to your repository/image. The container user must
        # be able to write to the sync target; set ownership in the dev image.
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: ./Dockerfile
        # Also add dependency manifests/lockfiles that require a rebuild.
```

Run development explicitly with both files so a conventionally named override cannot accidentally weaken a production deployment.

## `compose.production.yaml`

```yaml
services:
  app:
    # Example: registry.example.com/myapp@sha256:<64-character-digest>
    image: ${APP_IMAGE:?Set APP_IMAGE to an approved immutable image reference}
    pull_policy: always
    restart: always
    environment:
      APP_ENV: production
      LOG_LEVEL: info
    deploy:
      resources:
        limits:
          cpus: "1.00"
          memory: 512M
          pids: 200

  db:
    # Pin the reviewed PostgreSQL patch version or, preferably, digest.
    image: ${POSTGRES_IMAGE:?Set POSTGRES_IMAGE to an approved pinned image}
    pull_policy: always
    restart: always
    deploy:
      resources:
        limits:
          cpus: "2.00"
          memory: 1G
          pids: 300
```

The resource values are examples, not recommendations for every workload. Measure normal and peak behavior before setting them; an overly low memory limit can cause out-of-memory kills. Do not add replicas blindly: stateful services, fixed host ports, session handling, and reverse-proxy configuration all affect whether scaling is safe.

## `.env.example`

Commit only safe example values.

```dotenv
# Local development defaults
APP_IMAGE=myapp:local
POSTGRES_IMAGE=postgres:18
APP_BIND=127.0.0.1
APP_PORT=8080
APP_ENV=development
LOG_LEVEL=debug
POSTGRES_DB=app
POSTGRES_USER=app
DB_PASSWORD_FILE=./secrets/db_password.txt
```

Use a separate, protected `.env.production` for deployment. It should set at least:

```dotenv
APP_IMAGE=registry.example.com/myapp@sha256:REPLACE_WITH_REVIEWED_DIGEST
POSTGRES_IMAGE=postgres:18@sha256:REPLACE_WITH_REVIEWED_DIGEST
APP_BIND=127.0.0.1
APP_PORT=8080
POSTGRES_DB=app
POSTGRES_USER=app
DB_PASSWORD_FILE=/secure/path/db_password.txt
```

Changing `POSTGRES_USER`, `POSTGRES_DB`, or the password file does not rotate credentials in an already initialized PostgreSQL volume; those variables primarily initialize a new database. Plan credential rotation separately.

## `.gitignore`

```gitignore
.env
.env.*
!.env.example
secrets/
```

## `.dockerignore`

```dockerignore
.git
.env
.env.*
secrets/

```javascript

Add language-specific caches, test output, local data, and other files that are not needed in the build context. Do not bake secret files into an image.

## Setup and validation

```

# 1. Create local configuration and a development-only database password.

cp .env.example .env
umask 077
mkdir -p secrets
openssl rand -base64 32 > secrets/db_password.txt

# File-backed secrets are bind mounts. For this rootful local-Docker example,

# let the app's UID/GID own the file; the PostgreSQL entrypoint starts as root.

# Adapt ownership to your container users and rootful/rootless Docker setup.

sudo chown 10001:10001 secrets/db_password.txt
sudo chmod 0400 secrets/db_password.txt

# 2. Update the app health-check command and development watch paths.

# 3. Inspect the fully merged/interpolated development model.

docker compose -f compose.yaml -f compose.dev.yaml config

# 4. Start local development with build and file watching.

docker compose -f compose.yaml -f compose.dev.yaml up --build --watch

```javascript

For production:

```

# Validate and inspect the resolved model before pulling or changing services.

docker compose \
--env-file .env.production \
-f compose.yaml -f compose.production.yaml \
config

docker compose \
--env-file .env.production \
-f compose.yaml -f compose.production.yaml \
pull

docker compose \
--env-file .env.production \
-f compose.yaml -f compose.production.yaml \
up -d --wait

docker compose \
--env-file .env.production \
-f compose.yaml -f compose.production.yaml \
ps

# Remove containers/networks only when intended. Do not routinely add -v:

# -v deletes named volumes declared by this Compose project, including db_data.

docker compose \
--env-file .env.production \
-f compose.yaml -f compose.production.yaml \
down
```

Be cautious about storing `docker compose config` output as a public CI artifact: it reveals resolved configuration and will expose anything that was mistakenly supplied as an environment variable.

## Production review questions

- Are all application and third-party images approved, scanned, and pinned? Is there a controlled update and rollback process?
- Does every container run with the minimum user, capabilities, filesystem access, devices, and network access it needs?
- Are secrets injected only into services that require them, excluded from source/build contexts, readable only by intended users, and rotatable?
- Has the example PostgreSQL superuser been replaced for application access by a separate least-privileged role and secret? Are major-version upgrades planned and tested separately from routine deployments?
- Does each health check test actual readiness without creating excessive load or depending on an external service that could cause cascading failure?
- Does the application reconnect after database/network failures instead of relying only on startup ordering?
- Are persistent volumes covered by encrypted, off-host backups with tested restores and retention?
- Are CPU, memory, PID, disk, and log usage bounded and monitored? Are alerts actionable?
- Are graceful shutdown, database migrations, deployment, rollback, and disaster recovery rehearsed?
- Are TLS, firewall rules, host hardening, audit logging, and operating-system updates handled outside Compose?
- Has the final `docker compose config` output been reviewed, including all overrides and remote references?

## Official references

### Compose model, configuration, and development

- [Compose version and project name](https://docs.docker.com/reference/compose-file/version-and-name/)
- [Use Compose in production](https://docs.docker.com/compose/how-tos/production/)
- [Merge Compose files](https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/)
- [Compose `include` reference](https://docs.docker.com/reference/compose-file/include/)
- [Compose file watch](https://docs.docker.com/compose/how-tos/file-watch/)
- [Environment-variable best practices](https://docs.docker.com/compose/how-tos/environment-variables/best-practices/)
- [Variable interpolation and `.env` files](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)
- [`docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/)
- [`docker compose up`, including `--wait` and `--watch`](https://docs.docker.com/reference/cli/docker/compose/up/)

### Security, secrets, and networking

- [Manage secrets securely in Docker Compose](https://docs.docker.com/compose/how-tos/use-secrets/)
- [Compose container specification: `secrets`, `user`, `read_only`, `tmpfs`, `security_opt`, and shutdown settings](https://docs.docker.com/reference/compose-file/container-spec/)
- [Compose services: `container_name`, `pull_policy`, and `restart`](https://docs.docker.com/reference/compose-file/services/)
- [Networking in Compose, including internal networks](https://docs.docker.com/compose/how-tos/networking/)
- [Compose trust model and remote-reference pinning](https://docs.docker.com/compose/trust-model/)
- [Dockerfile building best practices: multi-stage builds, pinning, `.dockerignore`, and non-root users](https://docs.docker.com/build/building/best-practices/)

### Health, resources, logs, lifecycle, and storage

- [Control startup and shutdown order](https://docs.docker.com/compose/how-tos/startup-order/)
- [Compose health-check reference](https://docs.docker.com/reference/compose-file/workload-spec/#healthcheck)
- [Docker restart policies](https://docs.docker.com/engine/containers/start-containers-automatically/#use-a-restart-policy)
- [Compose Deploy Specification: resource limits](https://docs.docker.com/reference/compose-file/deploy/#resources)
- [Local file logging driver and rotation options](https://docs.docker.com/engine/logging/drivers/local/)
- [Docker volumes, lifecycle, backup, and restore](https://docs.docker.com/engine/storage/volumes/)
- [`docker compose down` and volume removal](https://docs.docker.com/reference/cli/docker/compose/down/)
- [PostgreSQL official image: initialization, secrets, `PGDATA`, and major upgrades](https://hub.docker.com/_/postgres)