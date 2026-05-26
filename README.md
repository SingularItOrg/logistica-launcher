# logistica-launcher

Orchestration repo for the Electoral Logistics System. Brings up the full
stack (database, broker, backend, workers, frontend) with a single
`docker compose` invocation.

> **Audience:** infrastructure / DevOps. Application-level docs live in
> [`logistica-backend`](../logistica-backend/logistica-backend/README.md) and
> [`logistica-frontend`](../logistica-frontend/README.md).

---

## Repo layout

```
logistica-launcher/
├── docker-compose.yml   # orchestration for the whole stack
├── .env.example         # env vars consumed by docker-compose.yml
├── .env                 # local (gitignored) — copied from .env.example
└── README.md
```

The launcher does not contain application code. It is a sibling of
`logistica-backend/` and `logistica-frontend/`, and its compose file
references both via relative paths (`../logistica-backend/...`,
`../logistica-frontend`). The expected on-disk layout is:

```
<workspace-root>/
├── logistica-backend/
│   └── logistica-backend/        # actual backend repo (nested one level)
├── logistica-frontend/
└── logistica-launcher/           # you are here
```

If any of these is missing or renamed, the compose build will fail with
`unable to prepare context`.

---

## Services

| Service    | Image / build                            | Host port | Internal hostname | Purpose                                                    |
| ---------- | ---------------------------------------- | --------- | ----------------- | ---------------------------------------------------------- |
| `postgres` | `postgres:16-alpine`                     | `5432`    | `postgres`        | Primary database. Data persists in the `postgres_data` volume. |
| `redis`    | `redis:7-alpine`                         | `6379`    | `redis`           | Celery broker + result backend.                            |
| `backend`  | `../logistica-backend/...` (target `dev`) | `8000`    | `backend`         | Django + DRF API. Runs `migrate` then `runserver`.         |
| `worker`   | same image as `backend`                  | —         | `worker`          | Celery worker.                                             |
| `beat`     | same image as `backend`                  | —         | `beat`            | Celery beat scheduler (`DatabaseScheduler`).               |
| `frontend` | `../logistica-frontend/Dockerfile.dev`   | `5173`    | `frontend`        | Vite dev server with HMR.                                  |

All services share a single user-defined bridge network (`logistica`).
Service-to-service traffic uses the internal hostnames in the table; host
browsers and external tools use `localhost:<host port>`.

### Healthchecks and ordering

`postgres` and `redis` declare healthchecks (`pg_isready` and `redis-cli
ping`). `backend` and `worker` wait for both via `condition:
service_healthy`. `beat` waits for `worker`; `frontend` waits for
`backend`. A `docker compose up` therefore brings the stack up in the
right order automatically.

### Volumes

- `postgres_data` (named volume) — Postgres data directory. Survives `down`; deleted by `down -v`.
- `../logistica-backend/logistica-backend/src` → `/app/src` — bind mount so backend code changes are picked up by `runserver` autoreload.
- `../logistica-frontend` → `/app` — bind mount for Vite HMR. A second anonymous volume on `/app/node_modules` shields the container's `node_modules` from the host's (avoids native-build mismatches when the host is Windows).

---

## Environment variables

The compose file reads variables from a sibling `.env` file. Every
variable has a default suitable for local development, so an empty `.env`
also works.

Variables are split into two groups in `.env.example`: those that
**should be overridden** in staging / prod (credentials, hostnames,
debug flags), and **optional** ones that ship with safe development
defaults (email, observability, tracking provider).

### Required overrides for staging / production

| Variable               | Default                 | Consumed by                | Notes                                                              |
| ---------------------- | ----------------------- | -------------------------- | ------------------------------------------------------------------ |
| `POSTGRES_USER`        | `postgres`              | `postgres`, all Django svc | Also baked into `DATABASE_URL`.                                    |
| `POSTGRES_PASSWORD`    | `postgres`              | `postgres`, all Django svc | Change in lockstep with the username if you alter the URL shape.  |
| `POSTGRES_DB`          | `logistica`             | `postgres`, all Django svc |                                                                    |
| `DJANGO_SECRET_KEY`    | `dev-only-not-secret`   | `backend`, `worker`, `beat` | **Must be replaced** with a randomly generated value.              |
| `DJANGO_DEBUG`         | `True`                  | `backend`                  | Set to `False` outside of local dev.                               |
| `DJANGO_ALLOWED_HOSTS` | `*`                     | `backend`                  | Comma-separated. Restrict to the public hostname in staging / prod. |
| `VITE_API_URL`         | `http://localhost:8000` | `frontend`                 | Public backend URL **as seen from the browser**, not the container. |
| `CORS_ALLOWED_ORIGINS` | `http://localhost:5173` | `backend`                  | Comma-separated. Add the public frontend URL when deploying.       |

> **Browser vs server perspective:** `VITE_API_URL` is the URL the
> *browser* will hit. Setting it to `http://backend:8000` only works if
> the request originates from a container on the `logistica` network —
> it will not work from your host browser.

### Optional — sensible defaults already applied

These are exposed through the compose so they can be overridden via
`.env`, but they ship with development-friendly defaults from
`config/settings/base.py`. They are commented out in `.env.example`.

| Variable                   | Default                                                    | Notes                                                                      |
| -------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------- |
| `FRONTEND_BASE_URL`        | `http://localhost:5173`                                    | Embedded in verification / password-reset email links.                     |
| `EMAIL_BACKEND`            | `django.core.mail.backends.console.EmailBackend`           | Console backend prints emails to stdout. Switch to SMTP in staging / prod. |
| `EMAIL_HOST`               | _(empty)_                                                  | SMTP host.                                                                 |
| `EMAIL_PORT`               | `587`                                                      |                                                                            |
| `EMAIL_HOST_USER`          | _(empty)_                                                  |                                                                            |
| `EMAIL_HOST_PASSWORD`      | _(empty)_                                                  |                                                                            |
| `EMAIL_USE_TLS`            | `True`                                                     |                                                                            |
| `DEFAULT_FROM_EMAIL`       | `no-reply@logistica.local`                                 |                                                                            |
| `TRACKING_PROVIDER`        | `apps.tracking.providers.mock.MockTrackingProvider`        | Dotted path to the active GPS provider. See ADR-0002 in the backend repo.  |
| `LOG_LEVEL`                | `INFO`                                                     | Root log level.                                                            |
| `CELERY_TASK_ALWAYS_EAGER` | `False`                                                    | Force synchronous Celery execution. Debug aid.                             |

---

## Bring-up from scratch

```bash
# 1. Clone the three repos as siblings.
git clone <backend-url>  logistica-backend
git clone <frontend-url> logistica-frontend
git clone <launcher-url> logistica-launcher

cd logistica-launcher

# 2. Configure environment.
cp .env.example .env
# (edit .env as needed — defaults are fine for local dev)

# 3. Build and start everything.
docker compose up -d --build
```

The first build pulls the Postgres/Redis images, compiles backend deps
into the Python image, and runs `npm ci` for the frontend. Expect 3–8
minutes on a cold cache.

Once up, hit:

- Frontend: <http://localhost:5173>
- Backend API: <http://localhost:8000/api/v1/>
- Django admin: <http://localhost:8000/admin/>

### Create a superuser

The compose runs migrations automatically on `backend` startup, but does
not create a superuser. Do it once:

```bash
docker compose exec backend python manage.py createsuperuser
```

---

## Day-to-day commands

```bash
# Status
docker compose ps

# Tail logs (all services / one service)
docker compose logs -f
docker compose logs -f backend

# Restart a single service (e.g. after editing tasks.py)
docker compose restart worker beat

# Run a one-off Django command
docker compose exec backend python manage.py shell
docker compose exec backend python manage.py makemigrations
docker compose exec backend pytest

# Open a psql shell against the running database
docker compose exec postgres psql -U postgres logistica

# Rebuild after a Dockerfile or requirements change
docker compose build backend worker beat
docker compose up -d

# Stop everything (keeps the database)
docker compose down

# Stop AND wipe the database volume
docker compose down -v
```

> **Note on the backend `Makefile`:** the backend repo ships a `Makefile`
> whose `make up` / `make down` targets call `docker compose` against its
> *own* sibling `docker-compose.yml`. That compose file is for backend
> development in isolation and does not include the frontend. Use the
> launcher compose for the full stack.

---

## Staging / production notes

The current compose targets local development. To deploy:

1. **Switch the backend build target** to `prod` (the `Dockerfile` already defines a `prod` stage that runs `gunicorn` as a non-root user).
2. **Remove the bind mounts** for `backend`, `worker`, `beat`, and `frontend` — they exist only for hot reload.
3. **Set the env overrides** flagged as required in the table above: `DJANGO_DEBUG=False`, `DJANGO_ALLOWED_HOSTS=<real hostname>`, real `CORS_ALLOWED_ORIGINS`, real `VITE_API_URL`, and SMTP credentials (`EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend` plus the `EMAIL_*` block).
4. **Inject real secrets** for `DJANGO_SECRET_KEY`, database credentials, and SMTP via the orchestrator's secret store. Do not bake them into `.env` checked into git.
5. **Build the frontend statically** instead of using `Dockerfile.dev`. The current frontend repo expects to be served by Vite dev server; a production image (`npm run build` → static assets on nginx / S3 + CloudFront) is still to be defined — coordinate with the frontend team before deploying.
6. **Persistent storage** for `postgres_data` must live on durable storage (EBS / managed RDS) rather than a Docker named volume on an ephemeral host.
7. **Beat scheduler** uses `django_celery_beat`'s `DatabaseScheduler`, so schedules live in the DB. No filesystem state to persist for `beat` itself.

> **Adding a new env var to the Django services:** the compose defines a
> shared `x-django-env` block at the top of the file. Add the variable
> there once and it propagates to `backend`, `worker`, and `beat` via
> the `<<: *django-env` merge. Service-specific overrides (e.g. `DJANGO_DEBUG`
> on `backend` only) go in the service's own `environment:` block, which
> is merged on top of the shared block.

A dedicated `docker-compose.prod.yml` (or a Helm chart / ECS task
definition) should be derived from this one rather than reusing it as-is.

---

## Troubleshooting

| Symptom                                                                 | Likely cause / fix                                                                                       |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `unable to prepare context: ../logistica-backend/...`                   | The backend or frontend sibling is missing or renamed. Check the workspace layout above.                  |
| Backend fails with `could not connect to server: postgres`              | Postgres failed its healthcheck. `docker compose logs postgres`. Usually a volume-permissions issue on a fresh disk. |
| Frontend HMR not picking up file changes on Windows                     | Vite needs polling on Windows bind mounts. The `Dockerfile.dev` already sets `CHOKIDAR_USEPOLLING=true`. If it still fails, ensure the host clock and container clock are in sync. |
| `5173 / 8000 / 5432 / 6379 already in use`                              | Another local service is using the port. Either stop it or change the host-side port in `docker-compose.yml` (`"8001:8000"`). |
| Celery worker doesn't pick up a task you just added                     | Restart it: `docker compose restart worker beat`. Workers don't autoreload.                              |
| Migrations applied but admin login fails with "no such user"            | Superuser was not created. Run `docker compose exec backend python manage.py createsuperuser`.            |
| `node_modules` errors after switching between host and container builds | Wipe the anonymous volume: `docker compose down && docker volume prune` and `docker compose up -d --build`. |
