# Containers and reproducible local environments

[Tutorial index](../README.md) · Related: [Linux processes](01-linux-processes-and-networking.md)

## Outcome and prerequisites

Package the Task Service consistently and understand which state belongs inside an image, a container, a volume, or configuration. You need Docker with the Compose plugin installed and running, the capstone source and lockfile, and permission to run a local database. These are optional learning prerequisites; this tutorial does not install Docker or modify the host.

An image is a layered artifact describing a filesystem and startup configuration. A container is a running or stopped instance with its own writable layer and isolation settings. A named volume persists independently of a container's writable layer. A bind mount exposes a host path and therefore creates a direct dependency on host permissions, contents, and path conventions.

Linux containers on Windows commonly run through a Linux VM or WSL-backed engine. Paths inside the container are Linux paths even when the source lives on a Windows drive. Python native extensions must match the target operating system and architecture; copying a Windows virtual environment into a Linux image is unreliable.

## Build a focused runtime artifact

Use a build stage for dependency preparation and a runtime stage containing only what the process needs. This reduces accidental inclusion of compilers, build caches, and development dependencies. Multi-stage Dockerfiles copy selected output between stages. [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/)

1. In a learning copy of the capstone application, add this `Dockerfile`. `python:3.12-slim-bookworm` is an explicit teaching runtime, not a permanent production pin. Confirm compatibility, then replace the tag with a verified digest for a release. This template requires a fully pinned, hash-checked `requirements.lock` including a reviewed Gunicorn version in addition to Django and psycopg. Generate that file through your chosen dependency-locking workflow first.

```dockerfile
FROM python:3.12-slim-bookworm AS build
ENV VIRTUAL_ENV=/opt/venv
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.lock /tmp/requirements.lock
RUN pip install --no-cache-dir --require-hashes -r /tmp/requirements.lock

FROM python:3.12-slim-bookworm AS runtime
WORKDIR /app
ENV PATH="/opt/venv/bin:$PATH"
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
RUN groupadd --system --gid 10001 app && useradd --system --uid 10001 --gid app app
COPY --from=build /opt/venv /opt/venv
COPY --chown=app:app manage.py ./
COPY --chown=app:app config ./config
COPY --chown=app:app tasks ./tasks
USER 10001:10001
EXPOSE 3000
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:3000", "--workers", "2", "--graceful-timeout", "25"]
```

This assumes the capstone layout has `config`, `tasks`, and `manage.py`. Copy any additional required templates or static assets deliberately. Native dependencies built from source may need build and runtime libraries beyond this template. `EXPOSE` documents a port; it does not publish it to the host. Gunicorn runs here inside Linux even when the Docker host is Windows. [Django with Gunicorn](https://docs.djangoproject.com/en/5.2/howto/deployment/wsgi/gunicorn/)

2. Add a `.dockerignore` so build context excludes secrets and host dependencies:

```text
.venv
__pycache__
*.pyc
.git
.env
.env.*
*.log
*.prof
backups
```

3. Run `docker build -t task-service:local .` from that application directory after creating the required lock. Expect dependency installation and image creation. A build requires network access unless dependencies and layers are already cached. Inspect failures before retrying; do not disable integrity or TLS checks to bypass them.

## Connect services deliberately

Within a Compose network, use service names such as `db` instead of `localhost` to reach other containers. From the Windows host, use the database's published loopback port. These are different connection strings for different network locations.

The following is a standalone local Compose teaching file. It includes an application image built above and a PostgreSQL 17 database. It is not an instruction to overwrite the capstone's own Compose file. The credentials are deliberately local-only and must not be reused in a deployed environment.

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: taskapp
      POSTGRES_PASSWORD: taskapp_local_only
      POSTGRES_DB: taskapp
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - task_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taskapp -d taskapp"]
      interval: 5s
      timeout: 3s
      retries: 10
  api:
    image: task-service:local
    environment:
      PGHOST: db
      PGPORT: "5432"
      PGDATABASE: taskapp
      PGUSER: taskapp
      PGPASSWORD: taskapp_local_only
      DJANGO_SECRET_KEY: local-container-lab-only-change-before-deploying
      DJANGO_DEBUG: "1"
      DEMO_MODE: "1"
    ports:
      - "127.0.0.1:3000:3000"
    depends_on:
      db:
        condition: service_healthy
    stop_grace_period: 30s
volumes:
  task_data:
```

Initialize the database schema using the capstone's schema procedure before testing task routes. PostgreSQL readiness alone does not mean application tables exist. The API must listen on an interface reachable inside the container, usually the container wildcard interface; a loopback-only listener will not accept published-port traffic.

Compose can wait for dependencies marked `service_healthy` during startup. This is not ongoing fault tolerance: the application still needs bounded connection handling when a dependency later becomes unavailable. [Compose startup order](https://docs.docker.com/compose/how-tos/startup-order)

## Verification and operational boundaries

With this file saved in a dedicated lab directory as `compose.yaml`, inspect `docker compose config --quiet`, then use `docker compose up -d`. Check `docker compose ps`, `docker compose logs --tail 50 api`, and the two health routes from the host. The expected result is a running API, a healthy database container, and successful health checks once required initialization is complete.

Inspect image configuration to confirm the runtime user is non-root. Confirm `.env`, backups, and source-control metadata are absent from the image. Test shutdown under an in-flight request and verify that the grace period exceeds the application's drain deadline. Resource limits must be chosen from measurements; arbitrarily tiny limits can make a correct application fail during startup or garbage collection.

Named volumes preserve data across ordinary container replacement. Commands that delete volumes destroy that persistence; do not use volume deletion as routine troubleshooting. A volume is not a backup because it remains exposed to accidental deletion, storage loss, and logical corruption. See [backup and recovery](06-reliability-backups-and-disaster-recovery.md).

For real secrets, use an appropriate secret delivery mechanism. Compose secrets are mounted as files and granted per service; the application or image must explicitly support reading them. Adding a secret declaration does not automatically make Django settings understand a `_FILE` variable. The settings above deliberately enable the capstone's local demo; disable it, disable debug output, configure hosts, and implement real authentication before any deployment. [Compose secrets](https://docs.docker.com/compose/how-tos/use-secrets)

## Tradeoffs, mistakes, and exercises

Development bind mounts support rapid edits but can conceal missing image files or introduce host-specific behavior. Verify the final image without development mounts. Minimal images reduce included software but may require separate debugging tools or an approved diagnostic container. Avoid installing troubleshooting packages into running production containers as an undocumented fix.

Common mistakes include baking credentials into layers, using mutable release tags without recorded digests, running as root unnecessarily, publishing database ports on every interface, and assuming container restarts repair corrupted data. Build arguments are not secret storage; use the builder's secret facility for private package access.

Practice rebuilding after changing only application code and observe which layers are reused. Then change the lockfile and observe dependency layers rebuild. Replace the API container while preserving the database volume and verify tasks remain. Apply migrations through one controlled job; do not have every application replica race to migrate on startup. Finally document image identity, configuration, ports, volumes, initialization, shutdown, and cleanup ownership so another developer can reproduce the environment.
