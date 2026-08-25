# CLAUDE.md

Bahasa Indonesia learning platform: a turn-based roguelike whose encounters are
resolved through Indonesian dialogue, plus structured lessons and per-user
progress. Django API + React/Phaser frontend.

## Status

**The repository is documentation-only right now.** There is no application
code — the commands below become live once task 1 in `_docs/tasks.md` is done.
Read `_docs/` before writing anything; the stack, architecture, and build order
are already decided and written down.

## Commands

```bash
uv sync                              # install dependencies
uv run pytest                        # the whole suite
uv run pytest tests/test_home.py     # one test file
uv run ruff check .                  # lint
uv run ruff format .                 # format

docker compose up -d                 # local Postgres 18
uv run python manage.py migrate      # apply migrations
uv run python manage.py makemigrations --check --dry-run   # CI runs this too
uv run python manage.py runserver
```

## Rules

- **Dependencies are added in `pyproject.toml`. Do not add one without asking.**
  The same applies to `package.json` on the frontend.
- **Versions are pinned deliberately in `_docs/ARCHITECTURE.md` §8.** Django is
  held at 5.2 LTS rather than the newer 6.1 for a stated reason. Do not upgrade
  anything without asking.
- **Game rules live on the server, in Python.** Encounter resolution, run state,
  scoring, and answer-checking are Django code. The client sends intent only.
  Never move rule logic into the frontend — forgeable progress is the failure
  mode this design exists to prevent.
- **Dialogue and lesson content live in the database, authored through the
  Django admin.** Never hardcode dialogue in Python or TypeScript, and do not
  add content as fixtures or migrations.
- **One migration-producing branch at a time**, and never edit a migration that
  has already been applied on `main`. Two branches each adding `0003_*` is the
  most common way to break this repo.
- **API changes are additive within a PR.** The frontend and API deploy as
  separate jobs and either can fail alone. Removing a field takes two PRs: one
  that stops using it, a later one that deletes it.
- **One task, one branch, one PR.** Branch prefixes and review weights are in
  `_docs/IMPLEMENTATION.md`.
- **Never commit `.env` or secrets.** Configuration comes from the environment.

## Where things are

| File | Contents |
|---|---|
| `_docs/plan.md` | Product vision, learner model, content model, roadmap |
| `_docs/ARCHITECTURE.md` | Topology, pinned versions, settings, what's deliberately unused |
| `_docs/tasks.md` | **The task backlog — single source of truth for what to build** |
| `_docs/IMPLEMENTATION.md` | Git strategy, branching conventions, working practices |
| `_docs/DEPLOY.md` | Runbook: manual Hostinger and Railway setup steps |
| `_docs/ACCOUNTS.md` | Accounts, costs, and signups the owner needs to obtain |
