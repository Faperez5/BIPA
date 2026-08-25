# Task Backlog

Each task is sized for one working session and written to be handed to someone
who has not read the others. Stack, versions, and settings referenced below are
specified in [ARCHITECTURE.md](ARCHITECTURE.md); the product they add up to is
described in [plan.md](plan.md).

Tasks are listed in a sensible build order and there are real dependencies
between them — nobody writes dialogue models before Django exists. But each
description stands on its own, so you can pick one up cold.

Branch and PR conventions: [IMPLEMENTATION.md](IMPLEMENTATION.md).

---

## 1. Empty project with a passing test
Goal: A clone-and-run Django project with one green test.
Description: Create a Django 5.2 project managed by `uv` with a `pyproject.toml`, configure `pytest` + `pytest-django` and `ruff` for lint and format, and add a single trivial test that asserts something true. The point is the scaffolding, not the test — a fresh clone should reach a green `pytest` run and a clean `ruff check .` with no manual setup steps beyond `uv sync`.

## 2. Local Postgres via Docker Compose
Goal: Django talks to a containerised Postgres in local development.
Description: Add a `compose.yaml` running `postgres:18` with a named volume, and point Django at it using the `psycopg` 3 driver (not `psycopg2`). Verify with `manage.py migrate` applying Django's built-in migrations successfully against the container.

## 3. Environment-based settings
Goal: Configuration comes from the environment, not hardcoded values.
Description: Introduce `django-environ` and move `SECRET_KEY`, `DEBUG`, `DATABASE_URL`, and `ALLOWED_HOSTS` into environment variables, with a committed `.env.example` documenting each. Local development reads a gitignored `.env` file; production will read real environment variables from the host.

## 4. CI test job
Goal: Every pull request runs lint and tests automatically.
Description: Replace the existing `.github/workflows/deploy.yml` with a `test` job that runs `ruff check`, `pytest` against a `postgres:18` service container, and `manage.py makemigrations --check --dry-run` so a model change without a matching migration fails the build. This project is partly a CI/CD practice vehicle, so the workflow should be readable and commented.

## 5. Custom user model
Goal: A project-owned user model exists before any user rows do.
Description: Add a custom user model (email as the login identifier rather than username) and set `AUTH_USER_MODEL` to point at it. This must land early — swapping the user model after real user rows exist is a difficult migration, which is the only reason this task is separated from the authentication work.

## 6. Track, Stage, and Lesson models
Goal: The course structure exists in the database.
Description: This is a Bahasa Indonesia learning platform where learners pick themed tracks ("Ordering food", "Getting a KTP") that each contain ordered stages of increasing difficulty, and each stage contains lessons. Model `Track`, `Stage`, and `Lesson` with explicit ordering fields, and write migrations plus a test that builds a track with two stages and three lessons.

## 7. VocabItem and GrammarPoint models
Goal: Reusable atomic language units, shared across the platform.
Description: Model `VocabItem` (Indonesian term, English gloss, notes) and `GrammarPoint` as standalone entities that lessons, dialogue, and game encounters all reference by foreign key rather than duplicating. Reuse is the whole point — the same word appears in several tracks, and shared references are what later allow "has this learner seen this word?" queries.

## 8. Dialogue node and choice models
Goal: Branching conversations stored as data, not code.
Description: In this platform, conversations with NPCs are the primary lesson delivery mechanism — the learner picks an Indonesian reply from several options. Model `DialogueNode` (one beat of NPC speech) and `DialogueChoice` (a player reply with its correctness, consequence, and next node), each with foreign keys to the `VocabItem` and `GrammarPoint` records it exercises, since those references are what makes automated content validation possible later.

## 9. Encounter model
Goal: A reusable template for one roguelike encounter.
Description: The game is a turn-based roguelike whose encounters are resolved through dialogue rather than combat. Model `Encounter` with an entry dialogue node, win and lose consequences, and the pool of vocabulary it draws from, so that individual encounters can be authored once and reused across randomised runs.

## 10. Run and UserProgress models
Goal: Per-user play state and completion are persisted.
Description: Model `Run` (one roguelike playthrough — its random seed, current state as a JSONB field, and whether it has ended) and `UserProgress` (per-user completion and score per lesson and encounter). Use a relational shape for anything queried across users and JSONB only for the nested run state, which is per-user and never queried by shape.

## 11. Django admin for course structure
Goal: Tracks, stages, and lessons are editable through a web UI.
Description: Register the `Track`, `Stage`, and `Lesson` models in Django admin with stages inlined on tracks and lessons inlined on stages, plus useful `list_display` and `search_fields`. All content on this platform is hand-authored by one person, so admin ergonomics directly determine how fast the course library can grow.

## 12. Django admin for vocabulary
Goal: Vocabulary and grammar are editable and easy to find.
Description: Register `VocabItem` and `GrammarPoint` in Django admin with search, filtering by track, and `autocomplete_fields` wherever other models reference them. Without autocomplete, selecting a word from a dropdown of thousands becomes the slowest part of authoring a lesson.

## 13. Django admin for dialogue authoring
Goal: A complete branching conversation can be written without touching code.
Description: Register `DialogueNode` and `Encounter` in Django admin with choices inlined on nodes and autocomplete for the vocabulary foreign keys. The success criterion is concrete and worth taking seriously: someone should be able to author a full multi-branch conversation, start to finish, without opening a shell or editing a fixture file.

## 14. Email and password authentication
Goal: Users can register, log in, and log out.
Description: Build signup, login, and logout using Django's built-in authentication against the project's custom user model, and configure `argon2-cffi` as the first entry in `PASSWORD_HASHERS` since Django defaults to PBKDF2. Password reset is not needed yet, but the user model should not make adding it later a migration problem.

## 15. Cross-subdomain session cookies
Goal: The session cookie works when the API and frontend are on sibling subdomains.
Description: In production the frontend is served from `games.yourdomain.com` and the Django API from `api.yourdomain.com`, so the session cookie must be scoped to `.yourdomain.com` to stay first-party — otherwise browsers that block third-party cookies will silently drop it. Configure `SESSION_COOKIE_DOMAIN`, `SESSION_COOKIE_SAMESITE`, `SESSION_COOKIE_SECURE`, `CSRF_TRUSTED_ORIGINS`, and `django-cors-headers` with credentials allowed for the frontend origin.

## 16. Production Dockerfile
Goal: The Django app runs as a container image.
Description: Write a multi-stage Dockerfile on a `python:3.13-slim` base that installs dependencies from the `uv` lockfile and serves via `gunicorn`, with `whitenoise` handling static files for the Django admin. Sync workers are appropriate here — the application is request/response with no long-lived connections.

## 17. Deploy the API to Railway
Goal: The Django app is live at `api.yourdomain.com` over HTTPS.
Description: Create a Railway project with a managed Postgres instance, attach the custom domain `api.yourdomain.com` via a CNAME from Hostinger's DNS, and add a `deploy-api` job to the GitHub Actions workflow that builds the image and runs `manage.py migrate` after deploying. The custom domain matters beyond aesthetics: a `*.up.railway.app` address would make the session cookie third-party and break login.

## 18. Gate the deploy jobs behind a flag
Goal: Deploy jobs exist but stay dormant until the site is worth showing.
Description: Add `if: vars.DEPLOY_ENABLED == 'true'` to every deploy job in the GitHub Actions workflow and leave that repository variable unset, so merges to `main` run tests without publishing anything to the live Hostinger subdomain. Set the variable only once there is a frontend worth serving; until then the deploy path stays exercised in configuration but inert in effect.

## 19. Content validation: vocabulary ordering
Goal: CI fails when a dialogue uses a word the learner has not been taught.
Description: Write a test that walks the authored content graph and asserts that every `VocabItem` referenced by a dialogue choice is introduced by an earlier lesson within the same track. This is the highest-value test in the project — it catches the pedagogical errors that are otherwise invisible until a learner hits them.

## 20. Content validation: structural integrity
Goal: CI fails on unreachable or orphaned content.
Description: Write tests that detect dialogue nodes with no path from any encounter entry point, encounters whose vocabulary pool is empty at their stage, lessons not attached to a stage, and stages containing no lessons. These are the mistakes that hand-authoring hundreds of interlinked content records reliably produces.

## 21. API framework and health endpoint
Goal: A typed HTTP API layer is wired up and reachable.
Description: Add `django-ninja` and expose a single health endpoint returning service and database status, with the auto-generated OpenAPI documentation rendering correctly. `django-ninja` was chosen over Django REST Framework for its Pydantic schemas and type hints, so the endpoint should demonstrate a typed response schema rather than a bare dict.

## 22. Content read endpoints
Goal: The frontend can fetch tracks, stages, and lessons over HTTP.
Description: Expose read-only `django-ninja` endpoints listing tracks, retrieving a track with its stages, and retrieving a lesson with its vocabulary, all with explicit response schemas. Keep them read-only — content is authored through the Django admin, never through the public API.

## 23. Authentication endpoints for the frontend
Goal: A JavaScript client can register, log in, and check session status.
Description: Expose signup, login, logout, and current-user endpoints that a browser client can call with `credentials: "include"`, returning appropriate status codes rather than HTML redirects. These sit alongside Django's session authentication rather than replacing it, so the admin login continues to work unchanged.

## 24. Start a run with a seeded random generator
Goal: A learner can begin a roguelike run that is reproducible.
Description: Write server-side logic that creates a `Run` for a user with an explicit random seed stored on the record, and initialises its starting state. Every downstream random decision must derive from that stored seed so any run can be replayed exactly — which matters enormously for debugging a game whose bugs are otherwise unreproducible.

## 25. Select an encounter from the learner's vocabulary
Goal: Runs present encounters matched to what the learner has studied.
Description: Write server-side logic that picks the next encounter for a run, drawing from encounters whose vocabulary the learner has already met in their chosen track, using the run's stored seed for the random choice. Keep it as a pure Python function taking state and returning a decision, with no HTTP or database writes, so it can be tested directly.

## 26. Resolve a dialogue choice
Goal: Picking a reply advances the run's state on the server.
Description: Write server-side logic that takes a run and a chosen `DialogueChoice`, validates that the choice is actually available from the run's current position, applies its consequence, and returns the new state. Validation is the important part — the client sends intent only, so that scores and progress cannot be forged from the browser console.

## 27. Record progress from run events
Goal: Playing the game updates the learner's completion record.
Description: Connect run outcomes to `UserProgress` so that resolving encounters and completing lessons updates per-user completion and scores, with rollups available per stage and track. Progress writes should happen server-side as a consequence of resolved turns, never from a client-supplied score.

## 28. Save and resume a run
Goal: A learner can close the browser mid-run and come back to it.
Description: Persist the run's in-progress state to its JSONB field after each resolved turn, and add an endpoint that returns a user's active run so a session can be picked up where it left off. Also handle run completion and permadeath, marking the run ended so it is no longer resumable.

## 29. Author the first track's lessons
Goal: One real track exists with genuine teaching content.
Description: Using the Django admin, author a complete "Survival Indonesian" track — stages, lessons, vocabulary, and grammar points — with enough material to support a full playthrough. This is content work rather than code, and it is also the first honest test of whether the authoring UI is fast enough to live with.

## 30. Author the first track's dialogue and encounters
Goal: The first track has playable conversations.
Description: Using the Django admin, write branching dialogue and between five and ten encounters for the "Survival Indonesian" track, drawing only on vocabulary that track's lessons introduce. The content validation tests should pass against it — if they do not, that is a genuine content bug worth fixing rather than a test to relax.

## 31. Throwaway dialogue interface
Goal: A full run is playable in a browser, unstyled.
Description: Build a minimal server-rendered interface using Django templates — dialogue text and a list of buttons, no styling budget, no JavaScript framework — that lets you play a complete run end to end. This is deliberately disposable and gets deleted once the real frontend exists; its only job is to make the learning loop playable early enough to evaluate honestly.

## 32. Evaluate the learning loop
Goal: A decision about whether the core loop actually works.
Description: Play the game daily for a week using the throwaway interface and write down whether it is teaching you anything and whether the encounter loop holds interest without art or animation. This is not a coding task and it is deliberately in the backlog: if the loop is weak, the fix belongs in content design or the encounter logic, and no amount of later frontend work will rescue it.

## 33. Frontend scaffold and deployment
Goal: A static site builds and deploys to the games subdomain.
Description: Create a Vite 7 + React 19 + TypeScript project that builds to `dist/`, and retarget the existing FTP deploy job in GitHub Actions from `public/` to that directory. The Hostinger subdomain and its four FTP secrets are already configured and should be reused as-is — only the source directory changes.

## 34. Frontend authentication
Goal: Users can log in from the static frontend to the API.
Description: Implement login, signup, and session handling in the React app, calling the Django API on a sibling subdomain with `credentials: "include"`. Test this explicitly in Safari and Firefox with third-party cookie blocking enabled — the cross-subdomain cookie is the highest-risk assumption in the whole architecture, and it fails silently when misconfigured.

## 35. Dialogue interface in React
Goal: The real frontend replaces the throwaway one.
Description: Rebuild the dialogue interface in React — NPC text, choice buttons, and inline vocabulary glosses — talking to the same API endpoints the server-rendered version used, then delete the Django templates it replaces. Keep the text UI as ordinary DOM rather than canvas, since a turn-based dialogue game is mostly reading and clicking.

## 36. Run loop interface
Goal: The roguelike structure is visible and playable in the frontend.
Description: Build the run interface in React — starting a run, moving between encounters, seeing run state, and handling permadeath and completion screens. All sequencing decisions come from the server; this task renders them and sends the learner's choices back.

## 37. Meta-progression between runs
Goal: Ending a run leaves something behind for the next one.
Description: Design and implement what carries across runs — unlocked vocabulary, NPC relationships, or persistent upgrades — as server-side state separate from any individual `Run` record. This is what makes a roguelike compelling rather than repetitive, and it is worth prototyping the rules before building interface for them.

## 38. Asset pipeline
Goal: Game art and audio ship with the frontend build.
Description: Configure Git LFS to track sprite, tileset, and audio sources, and extend the frontend build so processed assets are uploaded to the Hostinger subdomain alongside the bundle. Serving assets from the same origin as the page avoids both CORS configuration and paying for a separate CDN.

## 39. Phaser integration spike
Goal: An answer to whether Phaser and React can share a screen cleanly.
Description: On a throwaway branch, build the smallest possible test of a Phaser 4 canvas rendering underneath React DOM elements, specifically checking for z-index conflicts, pointer-event capture, and canvas resize behaviour. The output of this task is a written answer and a decision, not code to keep — the integration seam is the risky part, not the rendering.

## 40. Map view in Phaser
Goal: Runs are displayed on a visual map instead of a list.
Description: Build the run map as a Phaser 4 canvas layer beneath the existing React dialogue interface, using tilemaps exported from Tiled, showing the learner's position and the encounters ahead. Phaser owns only the map and encounter staging here — all text remains React DOM above it.

## 41. Progress dashboard
Goal: Learners can see how far they have come.
Description: Build a dashboard showing per-track and per-stage completion, vocabulary encountered, and run history, reading from the progress rollup endpoints. Keep it honest about partial progress — most learners will have several tracks part-finished rather than any one complete.

## 42. Require the CI check before merging
Goal: A failing test suite actually blocks a merge to `main`.
Description: The `main` branch is already protected — pull requests are required, force pushes and deletions are blocked, and linear history is enforced — but no status check is required yet, because requiring a check that does not exist would block every merge permanently. Once task 4 has created the `test` job and it has run at least once, add it as a required status check on `main` so that "CI must pass before merge" is enforced rather than merely documented. **Do this immediately after task 4**, out of numerical order.
