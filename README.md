# BIPA — Bahasa Indonesia learning games

Small vocabulary/grammar games for Bahasa Indonesia, built to accompany a real "Nivel 1" course. This project is primarily a **DevOps practice vehicle**: the game is the subject, but the point is exercising CI/CD, database migrations, containerized local dev, and deployment to a constrained shared-hosting target end to end.

## Architecture

- Course content (chapters, vocab, translations) is hand-curated as YAML in `content/`, reviewed against the course material, and loaded into a **MySQL** database that acts as the build-time source of truth.
- A small Python pipeline validates, loads, and exports that content to static `public/data/*.json`.
- The games themselves (flashcards, multiple-choice quiz, word-matching) are static HTML/CSS/vanilla JS, reading that JSON, with progress tracked client-side in `localStorage`.
- Production hosting (Hostinger shared) only ever receives finished static files — no persistent Python/PHP process runs there in v1.

## Build order

0. Repo scaffold, branching, PR/issue templates (this commit)
1. Docker Compose local MySQL + hand-rolled migration runner
2. Content pipeline: YAML → validate → load → export static JSON (+ pytest)
3. Static frontend: flashcards, multiple-choice quiz, word-matching, shared `localStorage` progress module
4. CI: lint/test, content validation, DB-integration tests against a MySQL service container
5. CD: build + deploy static artifacts to Hostinger via FTP/SFTP
6. *(stretch)* PHP `/api/progress.php` on Hostinger for cross-device progress
7. *(optional)* n8n glue automation for content ingestion

## Local development

Prerequisites: Docker, Python 3.12+.

```bash
make db-up       # start MySQL + adminer
make migrate     # apply SQL migrations
make load        # load content/*.yaml into MySQL
make export      # write public/data/*.json
make test        # run pytest
```

## Status

Early scaffolding — see the phase table in the plan doc for build order.
