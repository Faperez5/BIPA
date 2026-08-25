# Architecture & Stack Specification

*Companion to [plan.md](plan.md). This document pins concrete versions and
describes how the pieces connect. Versions verified 2026-08-23 — see
§8 Pin policy before changing any of them.*

---

## 1. Topology

Three hosts, one registrable domain. Keeping the existing Hostinger setup is
a deliberate choice: the AI Builder site and the FTP-deployed games subdomain
both stay exactly as DEPLOY.md describes, and Railway is added only for the
part Hostinger genuinely cannot do (a persistent Python process).

```
  yourdomain.com            Hostinger AI Website Builder
                            marketing / landing page, no-code, unchanged
                                      │ links to
                                      ▼
  games.yourdomain.com      Hostinger regular hosting (FTP)
                            static SPA bundle + game assets
                            deployed by the EXISTING deploy.yml FTP job
                                      │ fetch(), credentials: include
                                      ▼
  api.yourdomain.com        Railway (Docker container)
                            Django: auth, content API, admin CMS,
                            and server-authoritative game rules
                                      │
                                      ▼
                            Railway Postgres
                            content, dialogue, users, runs, progress
```

### Why this split works

`games.` and `api.` share the registrable domain `yourdomain.com`. A session
cookie scoped to `.yourdomain.com` is therefore **first-party**, so it is not
affected by the third-party cookie blocking now default in Safari and Firefox.
This is the detail that makes split hosting viable — if the API lived on
`*.up.railway.app`, the auth cookie would be third-party and would be dropped.

**Required for this to hold:**

| Setting | Value |
|---|---|
| Railway custom domain | `api.yourdomain.com` (CNAME from Hostinger DNS) |
| `SESSION_COOKIE_DOMAIN` | `.yourdomain.com` |
| `SESSION_COOKIE_SAMESITE` | `Lax` |
| `SESSION_COOKIE_SECURE` | `True` |
| `CORS_ALLOWED_ORIGINS` | `https://games.yourdomain.com` |
| `CORS_ALLOW_CREDENTIALS` | `True` |
| `CSRF_TRUSTED_ORIGINS` | `https://games.yourdomain.com` |
| Frontend `fetch()` | `credentials: "include"` |

Assets (sprites, tilesets, audio) are served from `games.yourdomain.com`
alongside the bundle — same origin as the page that loads them, so no CORS
headers needed for them and no separate CDN to pay for or configure.

---

## 2. Backend

| Component | Version | Notes |
|---|---|---|
| Python | **3.13** | Django 5.2 LTS supports 3.10–3.13. 3.14 exists but 3.13 has the broader package ecosystem. |
| Django | **5.2 LTS** | Supported to April 2028. See §8 for why LTS over the current 6.1. |
| API layer | **django-ninja 1.x** | Pydantic schemas and type hints — the FastAPI ergonomics you liked, inside Django. DRF 3.x is the alternative if you prefer the more conventional choice. |
| DB driver | **psycopg 3.2** | Not `psycopg2`. Django 5.x prefers psycopg 3. |
| Password hashing | **argon2-cffi** | Set `Argon2PasswordHasher` first in `PASSWORD_HASHERS`; Django defaults to PBKDF2 otherwise. |
| CORS | **django-cors-headers 4.x** | Configured per the table in §1. |
| App server | **gunicorn 23.x** | Sync workers are sufficient — turn-based means no long-lived connections. |
| Static files | **whitenoise 6.x** | For the Django admin's own CSS/JS only. The SPA is on Hostinger. |
| Env config | **django-environ 0.12** | Twelve-factor config from environment variables. |

### Server-authoritative game rules

Encounter resolution, run state, RNG seeding, and answer-checking are Django
code, not client code. The client sends *intent* (`POST /runs/{id}/choose`
with a choice id); the server validates it against the run's actual state and
returns the new state. Consequences:

- Progress and scores cannot be forged from the browser console.
- Game logic sits next to the content DB, so an encounter can query
  "VocabItems this learner has seen in this track" directly.
- The run seed lives server-side, so runs are reproducible and debuggable.

---

## 3. Frontend

| Component | Version | Notes |
|---|---|---|
| Node | **24 LTS** | Build toolchain only — nothing Node runs in production. |
| TypeScript | **5.x** | |
| Vite | **7.x** | Builds to a static `dist/` that the FTP job uploads. |
| React | **19** | App shell, routing, dialogue UI, dashboard. |
| Phaser | **4.1+** | Phaser 4 went stable April 2026 with a rewritten WebGL renderer. Do not start on Phaser 3. |
| Map editor | **Tiled 1.11** | Exports JSON tilemaps that Phaser loads directly. |

### The canvas/DOM division

Phaser owns the map view and encounter staging. Everything textual — dialogue,
choice buttons, vocab glosses, inventory, review-deck capture — is React DOM
rendered *over* the canvas. In a turn-based dialogue game this is the majority
of the interface, so Phaser is one component on a page, not the frame the app
lives inside.

This is the specific reason Phaser was chosen over Godot: a Godot web export is
a sealed canvas that cannot share DOM with the page, so every gloss and tooltip
would have to be rebuilt inside the engine.

---

## 4. Data layer

| Component | Version | Notes |
|---|---|---|
| PostgreSQL | **18.6** | Latest stable as of Aug 2026. Railway managed. |
| Migrations | Django built-in | No Alembic; `manage.py makemigrations`. |
| JSON storage | `JSONField` → JSONB | Run state and dialogue metadata. GIN-index what gets queried. |

Relational tables for anything you author or query across (tracks, stages,
lessons, vocab, grammar, dialogue nodes/choices, encounters). JSONB only for
run state, which is per-user, nested, and never queried by shape.

---

## 5. Local development

| Component | Version |
|---|---|
| Docker Engine + Compose | **v2** (`docker compose`, not `docker-compose`) |
| App image base | `python:3.13-slim` |
| DB image | `postgres:18` |
| Dependency manager | **uv** — fast, lockfile-based, replaces pip + venv + pip-tools |

`compose.yaml` runs Django + Postgres. The frontend runs on the host via
`npm run dev` with Vite proxying `/api` to the container, so the browser sees
one origin locally and the CORS config in §1 is only exercised in production.

---

## 6. Quality tooling

| Tool | Version | Purpose |
|---|---|---|
| ruff | latest, **pinned exactly** | Lint + format, replaces black/flake8/isort. Moves fast; unpinned versions break CI. |
| pytest + pytest-django | 8.x / 4.x | |
| factory-boy | 3.x | Test fixtures for content graphs |
| mypy | 1.x | Optional; django-ninja's Pydantic schemas give most of the benefit already |
| Biome or ESLint + Prettier | latest | Frontend lint/format |
| Git LFS | 3.x | Sprites, tilesets, audio |

---

## 7. CI/CD

`.github/workflows/deploy.yml` grows from one job to three:

| Job | Trigger | Does |
|---|---|---|
| `test` | every push + PR | ruff, pytest against a `postgres:18` service container, **content validation** (see below) |
| `deploy-api` | push to `main`, after `test` | Build Docker image, deploy to Railway, run `manage.py migrate` |
| `deploy-frontend` | push to `main`, after `test` | `npm run build`, FTP `dist/` + assets to Hostinger — **the existing job, retargeted from `public/` to `dist/`** |

### Content validation

The most valuable test in the suite, and the reason the plan kept a CI content
step at all. It walks the authored content graph and fails the build on:

- a `DialogueChoice` using a `VocabItem` not taught earlier in that track
- a dialogue node with no reachable path from an encounter entry point
- an `Encounter` whose vocab pool is empty at its stage
- orphaned lessons, stages with no lessons, broken FK references

### Secrets

Existing (keep): `HOSTINGER_FTP_SERVER`, `HOSTINGER_FTP_USERNAME`,
`HOSTINGER_FTP_PASSWORD`, `HOSTINGER_FTP_PATH`.
New: `RAILWAY_TOKEN`, `DJANGO_SECRET_KEY`, `DATABASE_URL`.

---

## 8. Pin policy

**Django 5.2 LTS over the current 6.1.** Django 6.1 shipped August 2026 and is
supported only to December 2027. 5.2 LTS runs to April 2028 and has the widest
third-party package compatibility. Nothing in this project needs 6.x features.
Django is also changing to an annual release cycle from January 2028, where
every release gets LTS-length support and versions are named by year
(Django 2028, 2029…) — 6.2 LTS, due ~April 2027, is the natural next hop.

**Pin exact versions for tooling, ranges for libraries.** `ruff` and frontend
tooling get exact pins because they change behaviour between patches and will
break CI silently. Application libraries get compatible-release ranges
(`~=`, `^`), resolved by `uv.lock` and `package-lock.json`, both committed.

**Verify before pinning.** Every version here was correct on 2026-08-23 but
Phaser, ruff, and Vite in particular move quickly. Re-check at the moment you
write the lockfiles rather than trusting this table.

---

## 9. Deliberately not used

| | Why |
|---|---|
| **MySQL** | Replaced by Postgres — JSONB for run state, and it's the first-class managed DB on Railway. |
| **Pyodide / pygbag** | Server-side Python gives gameplay-logic-in-Python without a multi-MB WASM download or mobile Safari fragility. |
| **Godot** | Sealed canvas, no DOM sharing. See §3. |
| **Redis** | No caching or queue need yet. Add it when there's a measured reason. |
| **Celery** | No background jobs in scope. |
| **A separate CDN** | Assets ship to Hostinger with the bundle, same origin. |

---

## 10. Unresolved

- Railway region — pick the one closest to your learners (Singapore for
  Indonesia-based users) since every turn is a round trip.
- Whether the SPA needs routing beyond a handful of views, or React Router.
- Art direction and asset sourcing (also open in plan.md §9) — gates Phase 2.
- Backup strategy for Postgres beyond Railway's built-in snapshots.
