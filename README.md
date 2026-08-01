# ⛪ FaithFlows

A multi-tenant church management platform. Each church is a tenant with its own
PostgreSQL schema, reached at its own subdomain.

This repository is a thin shell: the application lives in two submodules.

| Path | What it is | Repository |
| --- | --- | --- |
| `faithflow-backend/` | Django REST API | [churchcms-be-v2](https://github.com/NatMaestro/churchcms-be-v2) |
| `faithflow-studio/` | React web client | [faithflow-studio](https://github.com/NatMaestro/faithflow-studio) |

---

## Tech stack

**Backend** — Django 4.2, Django REST Framework, `django-tenants` (schema per
church), PostgreSQL (Neon), SimpleJWT, drf-spectacular. Python 3.11.

**Frontend** — React 18, TypeScript, Vite, Redux Toolkit, React Router v6,
shadcn/ui on Tailwind, Axios.

**Deployment** — the backend runs on Render under `config.settings_production`;
the frontend builds to static assets served from S3 behind CloudFront. Both
deploy from `main` via GitHub Actions.

---

## Getting started

```bash
# The application code is in submodules — a plain clone leaves them empty.
git clone --recurse-submodules <repository-url>
cd faithflows

# Already cloned without them?
git submodule update --init
```

### Backend

```bash
cd faithflow-backend
python -m venv venv && source venv/Scripts/activate   # Linux/macOS: venv/bin/activate
pip install -r requirements.txt

cp env.example .env        # then set DATABASE_URL and SECRET_KEY

python manage.py migrate_schemas --shared
python manage.py migrate_schemas
python manage.py runserver                            # http://localhost:8000
```

### Frontend

```bash
cd faithflow-studio
npm install

cp env.example .env        # VITE_API_URL, e.g. http://localhost:8000/api/v1

npm run dev                                           # http://localhost:8080
```

`VITE_API_URL` is inlined at **build** time, not read at runtime — setting it on
an already-built container has no effect.

### Multi-tenancy locally

Tenants are selected by the `X-Tenant-Subdomain` header, which the client
derives from the hostname. To exercise that locally, add entries to your hosts
file and browse to the subdomain rather than bare `localhost`:

```
127.0.0.1 olamchurch.localhost
127.0.0.1 gracechurch.localhost
```

Then open `http://olamchurch.localhost:8080`.

---

## Checks

Both submodules run CI on every pull request.

```bash
# faithflow-backend
python manage.py check
python manage.py makemigrations --check --dry-run
python manage.py test tests.test_tenant_binding tests.test_subscription_webhook \
                      tests.test_queryset_scoping tests.test_role_permissions

# faithflow-studio
npm run verify        # ratcheted typecheck + lint, then build
```

The frontend carries a known backlog of TypeScript and ESLint errors, recorded
in `quality-baseline.json`. CI fails on any **increase**, so the backlog does
not block work while new problems still do. When you drive a count down, lock it
in with `npm run ratchet:typecheck -- --update`.

The backend test suites listed above need no database. The rest of `tests/`
requires a provisioned tenant schema against a live PostgreSQL and is not yet
wired into CI.

---

## Configuration worth knowing

| Variable | Where | Notes |
| --- | --- | --- |
| `DATABASE_URL` | backend | PostgreSQL connection string. Required in production. |
| `SECRET_KEY` | backend | Required in production; the development default is a known literal. |
| `SETUP_TOKEN` | backend | Guards the unauthenticated initial-setup endpoint. Must be 32+ characters, or that endpoint stays disabled. |
| `PAYSTACK_SECRET_KEY` | backend | Also verifies webhook signatures — without it, webhooks are rejected. |
| `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` | backend | Password reset emails are the only delivery path for reset tokens. Unset means resets never reach users. |
| `VITE_API_URL` | frontend | Build-time only. |

---

## Features

**Members** — dashboard, profile, online giving and history, event RSVP,
announcements, ministries and small groups, volunteering, prayer requests and
altar calls, service requests (baptism, marriage, and so on).

**Church administrators** — member management and imports, event planning,
financial tracking and tax receipts, announcements, ministry and volunteer
management, role and permission management, theming, church settings.

**Platform administrators** — tenant management, denomination configuration,
subscription and trial control, feature flags.

Feature availability varies by denomination and by the church's plan; see
`Church.get_default_features()` in `faithflow-backend/apps/churches/models.py`.

---

## Documentation

- [`documentation/`](documentation/) — architecture and integration notes
- [`faithflow-backend/documentation/`](faithflow-backend/documentation/) — API and backend detail
- Interactive API reference at `/api/docs/` on a running backend

Both documentation folders have accumulated point-in-time status reports
alongside genuine reference material; treat undated "COMPLETE"/"FIX" write-ups
as historical.

---

## Contributing

1. Branch from `main`
2. Make your change, with tests where the behaviour is testable
3. Confirm CI passes
4. Open a pull request

---

## License

Not yet specified.
