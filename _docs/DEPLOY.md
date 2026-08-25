# Deployment Runbook

*The manual, one-time setup steps behind the architecture in
[ARCHITECTURE.md](ARCHITECTURE.md). That document explains the design and holds
the configuration values; this one is the click-through procedure for the
Hostinger and Railway control panels.*

## Hosting overview

Three pieces, one registrable domain:

| Address | Host | Serves |
|---|---|---|
| `yourdomain.com` | Hostinger AI Website Builder | Marketing / landing page |
| `games.yourdomain.com` | Hostinger regular hosting, over FTP | Static frontend bundle + game assets |
| `api.yourdomain.com` | Railway | Django: API, auth, admin, game rules |

The AI Website Builder is a no-code product with no File Manager, FTP, SSH, or
Git integration, so it cannot host the application — hence the subdomain under
the regular hosting plan, which does support FTP.

Hostinger's regular hosting is FTP and static-file oriented and cannot run a
persistent Python process, which is why the Django API lives on Railway. Both
subdomains sit under the same registrable domain so the session cookie stays
first-party — see ARCHITECTURE.md §1, and the troubleshooting note below.

hPanel's own Git auto-deploy (Advanced → Git) is deliberately **not** used: it
pulls raw files as-is and cannot run the frontend build step. GitHub Actions
builds and then uploads over FTP instead.

---

## Part 1 — Hostinger, in hPanel

1. Under the regular hosting plan, create the subdomain
   `games.yourdomain.com` and note its document root path.
2. Create an FTP/SFTP user **scoped to that document root only** — not to the
   account root.
3. Under DNS, add a `CNAME` record for `api` pointing at the target Railway
   gives you in Part 2 step 3. Do Part 2 first if that value isn't in hand yet.
4. In the AI Website Builder, add a nav link or Embed block pointing at
   `games.yourdomain.com`.

## Part 2 — Railway

1. Create a project from the repository and add a **Postgres 18** database;
   Railway injects `DATABASE_URL` automatically.
2. Set the service environment variables: `DJANGO_SECRET_KEY`, `DEBUG=False`,
   `ALLOWED_HOSTS=api.yourdomain.com`, and the cookie and CORS values from
   ARCHITECTURE.md §1.
3. Add the custom domain `api.yourdomain.com` and copy the CNAME target it
   shows into Part 1 step 3.
4. Pick the region closest to your learners — every turn of the game is a round
   trip, so this is a real latency decision, not a formality.

**Do not skip the custom domain.** On a default `*.up.railway.app` address the
session cookie becomes third-party and login fails silently in Safari and
Firefox.

## Part 3 — GitHub repository settings

Secrets (Settings → Secrets and variables → Actions):

| Secret | For |
|---|---|
| `HOSTINGER_FTP_SERVER` | Frontend deploy |
| `HOSTINGER_FTP_USERNAME` | Frontend deploy |
| `HOSTINGER_FTP_PASSWORD` | Frontend deploy |
| `HOSTINGER_FTP_PATH` | Document root from Part 1 step 1 |
| `RAILWAY_TOKEN` | API deploy |
| `DJANGO_SECRET_KEY` | API deploy |

Variables:

| Variable | Value |
|---|---|
| `DEPLOY_ENABLED` | Leave **unset** until there is a frontend worth publishing. Both deploy jobs are gated on it; tests still run on every pull request. |

---

## What happens on merge to `main`

`.github/workflows/deploy.yml` runs three jobs:

| Job | Does |
|---|---|
| `test` | ruff, pytest against a `postgres:18` service container, migration check. Runs on every push and PR. |
| `deploy-api` | Builds the Docker image, deploys to Railway, runs `manage.py migrate`. Gated on `DEPLOY_ENABLED`. |
| `deploy-frontend` | Runs the Vite build and uploads `dist/` to the Hostinger subdomain over FTP. Gated on `DEPLOY_ENABLED`. |

The two deploy jobs are independent and either can fail alone, leaving a new
frontend against an old API. Keep API changes additive within a single PR:
removing a field takes two PRs, one that stops using it and a later one that
deletes it.

---

## Troubleshooting

**Login works locally but not in production, and only in some browsers.**
Almost always the cookie domain. Confirm the API is reachable at
`api.yourdomain.com` and not a `*.up.railway.app` address, and that
`SESSION_COOKIE_DOMAIN` is `.yourdomain.com` with a leading dot. Test in Safari
or Firefox specifically — Chrome is the most forgiving and will hide this.

**Frontend deploys but shows stale content.** The FTP action uploads changed
files rather than clearing the directory. Check that the Vite build actually
produced new hashed filenames.

**Migrations did not run.** `deploy-api` runs `migrate` after the deploy, so a
failure there leaves new code against an old schema. Check the job log before
assuming an application bug.
