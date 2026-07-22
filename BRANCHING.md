# Branching strategy

Trunk-based, since this is a solo project — no perpetual `dev` branch.

- **`main`** — always deployable. Protected: no direct pushes, PRs required, CI must pass.
- **`feature/<name>`** or **`fix/<name>`** — short-lived, branched from `main`, merged back via PR.

```bash
git checkout main
git pull
git checkout -b feature/content-pipeline

# ...work, commit...

git push -u origin feature/content-pipeline
# open PR: feature/content-pipeline -> main, wait for CI, merge, delete branch
```

Releases aren't versioned/tagged for this project — every merge to `main` triggers a deploy (see `.github/workflows/deploy.yml`).
