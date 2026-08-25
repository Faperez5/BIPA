# Belajar Bahasa

A Bahasa Indonesia learning platform for expats and culturally-curious English
speakers. Learners pick themed tracks — "Ordering food", "Getting a KTP",
"Small talk with tetangga" — and work through lessons and a **turn-based
roguelike** whose encounters are resolved through Indonesian dialogue rather
than combat. The language is the mechanic, not a wrapper around one.

It is also a deliberate DevOps exercise: containerised local development,
migrations, content validation in CI, and deployment across two hosts.

## Status

**Documentation only — no application code yet.** The stack, architecture, and
build order are decided and written down. Start at
[`_docs/tasks.md`](_docs/tasks.md) task 1.

## Stack

Django 5.2 + Postgres 18 on Railway, serving a React 19 + Phaser 4 frontend
hosted on Hostinger. Game rules run server-side in Python; lesson and dialogue
content lives in the database and is authored through the Django admin.

Full detail and pinned versions: [`_docs/ARCHITECTURE.md`](_docs/ARCHITECTURE.md).

## Documentation

| Document | Contents |
|---|---|
| [`_docs/plan.md`](_docs/plan.md) | Product vision, learner model, content model |
| [`_docs/ARCHITECTURE.md`](_docs/ARCHITECTURE.md) | Topology, pinned versions, settings |
| [`_docs/tasks.md`](_docs/tasks.md) | The task backlog — what to build, in order |
| [`_docs/IMPLEMENTATION.md`](_docs/IMPLEMENTATION.md) | Branching and working practices |
| [`_docs/DEPLOY.md`](_docs/DEPLOY.md) | Hostinger and Railway setup runbook |
| [`_docs/ACCOUNTS.md`](_docs/ACCOUNTS.md) | Accounts, costs, and where credentials live |

## Local development

Prerequisites: Docker, Python 3.13+, `uv`. Available once task 1 is complete.

```bash
uv sync                              # install dependencies
docker compose up -d                 # start Postgres
uv run python manage.py migrate      # apply migrations
uv run pytest                        # run the suite
```
