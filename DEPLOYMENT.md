# Deployment

Production stack: PostgreSQL, Django/gunicorn backend, and an nginx-served
React SPA that reverse-proxies `/api` (and `/admin`, `/static`, `/media`)
through to the backend so the browser sees everything as one origin -
which is what makes the backend's `SameSite=Strict` auth cookies usable by
the SPA without any cross-site cookie relaxation.

```
.
├── docker-compose.yml
├── .env.example
├── pentest_reporting/            # Django backend (Dockerfile + entrypoint.sh)
└── pentest_reporting_frontend/   # React frontend (Dockerfile + nginx.conf)
```

## Run it

```bash
cp .env.example .env
# edit .env: set DB_PASSWORD, DJANGO_SECRET_KEY, CORS_ALLOWED_ORIGINS, etc.

docker compose up -d --build
```

- `db` (Postgres 16) starts first; `backend` waits on its healthcheck.
- `backend`'s `entrypoint.sh` polls the DB port, then runs `migrate`,
  `collectstatic`, and `seed_vulnerabilities` before starting gunicorn on
  `:8000` (internal only - not published to the host).
- `frontend` waits on the backend's `/healthz/` healthcheck, then serves
  the built SPA via nginx on `${FRONTEND_PORT:-80}`, proxying API/auth/
  static/media requests to `backend:8000` over the internal `pentest_net`
  bridge network.

Visit `http://localhost` (or your configured `FRONTEND_PORT`) once all
three services report healthy:

```bash
docker compose ps
```

## Creating the first admin/reviewer user

```bash
docker compose exec backend python manage.py createsuperuser
```

Superusers are treated as Admin/Reviewers by the RBAC checks regardless of
their `Profile.role` value. To promote a regular account instead, use the
Django admin at `/admin/` or the in-app Team page once you have at least
one Admin/Reviewer logged in.

## Cookies, HTTPS, and `DJANGO_COOKIE_SECURE`

Auth tokens are issued as HTTP-only, `SameSite=Strict` cookies. Browsers
refuse to send `Secure` cookies over plain HTTP, so:

- **Real deployments**: put this stack behind TLS (a reverse proxy /
  load balancer terminating HTTPS in front of the `frontend` container is
  the usual pattern) and leave `DJANGO_COOKIE_SECURE=true`.
- **Local-only HTTP testing**: set `DJANGO_COOKIE_SECURE=false` in `.env`
  so cookies still get set over `http://localhost`. Do not ship this
  setting to production.

## Volumes

- `pentest_db_data` - Postgres data directory (persists across restarts).
- `pentest_media` - uploaded client logos and PoC evidence screenshots.
- `pentest_static` - collected Django static files (served by whitenoise
  inside the backend container; also reachable via the frontend's
  `/static/` proxy path for the Django admin's own assets).
