# Working Practices

*How to run the build described in [tasks.md](tasks.md), using the stack pinned
in [ARCHITECTURE.md](ARCHITECTURE.md).*

**The task list lives in [tasks.md](tasks.md) and nowhere else.** This document
covers only how to work through it.

---

## Git strategy

Trunk-based, since this is a solo project — no perpetual `dev` branch.

- **`main`** — always deployable. Protected: no direct pushes, pull requests
  required, CI must pass before merge.
- **Everything else** — short-lived, branched from `main`, merged back via PR,
  then deleted.

```bash
git checkout main
git pull
git checkout -b feature/dialogue-schema

# ...work, commit...

git push -u origin feature/dialogue-schema
# open PR -> main, wait for CI, merge, delete branch
```

Releases are not versioned or tagged. Once `DEPLOY_ENABLED` is set, every merge
to `main` deploys — see [DEPLOY.md](DEPLOY.md).

---

## One task, one branch, one PR

| Prefix | For | Review weight |
|---|---|---|
| `feature/` | New capability | Full — CI green, self-review the diff |
| `fix/` | Bug fix | Full |
| `content/` | Lesson, dialogue, and vocabulary authoring | Light — content validation only |
| `chore/` | Tooling, dependencies, docs, config | Light |
| `ci/` | Workflow and deploy changes | Full — these break silently |
| `spike/` | Throwaway exploration | None — never merged as-is |

Name the branch after the task: task 8 becomes `feature/dialogue-schema`,
task 30 becomes `content/survival-indonesian-dialogue`.

---

## Ordering

Tasks in [tasks.md](tasks.md) are listed in build order, and each description
stands alone so it can be picked up cold. Real dependencies still exist:

- **Tasks 1–4** are the foundation everything else assumes.
- **Task 5 (custom user model) must land before task 14 (authentication).**
  Swapping `AUTH_USER_MODEL` once real user rows exist is a difficult migration.
- **Tasks 11–13 (admin) depend on 6–10 (models).** The admin is only worth
  building once there is something to author.
- **Task 32 is a gate, not a step.** Do not start 33 until it is answered.
- **Tasks 29–30 (content authoring) can run in parallel with anything**, and
  are the natural thing to do when you want a session that isn't code.

Two tasks decide whether this project succeeds: **13**, which sets your content
velocity for years, and **24–28**, which are the actual game. Everything else
is scaffolding around those.

---

## Migration discipline

The sharpest recurring hazard in this stack. Two branches that each add a
migration both produce `0003_*`, and merging leaves two leaf nodes.

- **One migration-producing branch at a time.** Tasks 5–10 are serial for this
  reason; resist the temptation to parallelise them.
- Rebase on `main` before opening a PR, then re-run `makemigrations --check`.
- CI runs `makemigrations --check --dry-run` (task 4), so a model change
  without a migration fails the build.
- On a collision, prefer deleting your local migration and regenerating on top
  of `main` over `makemigrations --merge`.

## Two deploy targets can drift

`deploy-api` (Railway) and `deploy-frontend` (Hostinger FTP) are separate jobs
and either can fail alone, leaving a new frontend against an old API.

**Rule: API changes are additive within a PR.** To remove a field, ship one PR
that stops using it, then a later PR that deletes it. Never both at once.

## Deploy gating

`deploy.yml` as it stands today FTPs to the live Hostinger subdomain on every
push to `main`, which would publish a broken site from the very first merge.
**Task 4 is what actually removes that**, replacing the workflow with a
test-only job — which is why it sits so early despite CI feeling like
later-stage work. Task 18 then reinstates gating when the deploy jobs come
back, so the deploy path is configured but inert until you set
`DEPLOY_ENABLED`.

## Content and code move at different speeds

Content pull requests will outnumber code ones roughly ten to one. Path-filter
CI so `content/` branches run validation and skip the frontend build, and keep
them small — one track or one encounter set per PR — so a validation failure
points somewhere specific.

## Spikes

`spike/` branches answer questions, not build features. Task 39 asks *can
Phaser render under React's DOM without z-index or input-capture fights?* —
answer it, throw the code away, then write the real thing (task 40) on a
`feature/` branch. Spike code is optimised for finding out, not for living with.

## Reviewing your own pull requests

PRs into a protected `main` are worth it even alone: CI gates the merge, the
diff view catches what the editor hides, and the PR body is where you record
*why*. Read your own diff before merging. Over roughly 400 lines, it should
probably have been two tasks.
