# Belajar Bahasa — Platform Plan
*Indonesian-for-expats learning site: lessons, games, and culture, built on top of the existing games-subdomain infrastructure*

## 1. Vision

Turn the current BIPA games subdomain into a self-serve Bahasa Indonesia learning
platform for expats and culturally-curious English speakers. Learners pick
**thematic tracks** (Survival, Work, Culture, Travel, etc.), each with an internal
difficulty progression, mixing **structured lessons** and **games**, with an
**account** that tracks progress across both.

This is a step up from the current architecture: today's pipeline produces
static JSON consumed by stateless games. Accounts + progress require a live
backend and database, not just a nightly export.

---

## 2. Learner model

- **No forced level test.** Learners self-select into thematic tracks (e.g.
  "Getting a KTP", "Ordering food", "Small talk with tetangga", "Understanding
  gotong royong culture").
- Each track has **internal difficulty progression** — lesson 1 of "Ordering
  food" is easier than lesson 5 of the same track — rather than a global CEFR
  ladder. This matches how expats actually learn: need-driven, not exam-driven.
- Tracks can share vocabulary/grammar building blocks under the hood (see §4)
  so progress in one track can unlock or reinforce another, even without an
  explicit CEFR label.

## 3. Content model

| Entity | Description |
|---|---|
| **Track** | Themed collection (e.g. "Culture & Etiquette"), has ordered stages |
| **Stage** | A difficulty step within a track (loosely: beginner/intermediate/advanced *for that track*) |
| **Lesson** | Explanatory content + vocab/grammar points, belongs to a stage |
| **Game** | Practice activity tied to a lesson or stage (flashcards, quiz, matching, or custom) |
| **VocabItem / GrammarPoint** | Shared atomic units, reusable across lessons/games/tracks |
| **UserProgress** | Per-user completion + score per lesson/game, rollup per track/stage |

Because you're authoring content manually into the DB/CMS, the schema should
optimize for **your** editing speed (good FK reuse of VocabItem/GrammarPoint
across tracks) more than for automated ingestion.

## 4. Architecture changes vs. current DEPLOY.md setup

### What stays the same
- GitHub Actions pipeline concept (`make migrate`, `make load`, `make test`, `make export`)
  is still valuable for **content authoring → validation → publish**, especially
  since you're hand-writing content and want a repeatable review step (tests
  catching broken cross-references between lessons/games/vocab).
- Hostinger AI Website Builder keeps being the marketing/landing site, linking
  to the subdomain, same as today.

### What has to change
- **Static JSON export is no longer sufficient.** Accounts and progress are
  read/write, per-user, at request time — that needs a live API + database,
  not a build artifact.
- The games subdomain becomes a real **web app** (frontend + backend API),
  not a static bundle of FTP'd files.
- MySQL likely stays as the content database (tracks/lessons/vocab), but you
  now also need a **users/progress** store — can be the same MySQL instance,
  just new tables (`users`, `user_progress`, `sessions`), no need for a
  separate DB technology.

### Recommended shape

```
[Hostinger AI Site]  →  links to  →  [games.yourdomain.com]
                                            │
                                    ┌───────┴────────┐
                                    │   Frontend      │  static/SPA, CDN-served
                                    │  (React/Vite)   │
                                    └───────┬────────┘
                                            │ REST/JSON
                                    ┌───────┴────────┐
                                    │   Backend API   │  Python (FastAPI)
                                    │  auth + progress │
                                    │  + content serve │
                                    └───────┬────────┘
                                            │
                                    ┌───────┴────────┐
                                    │     MySQL       │  content + users/progress
                                    └────────────────┘
```

Content authoring keeps your current Python/MySQL pipeline (`make migrate` /
`make load` / `make test`) as the **admin-side** workflow — you write lessons
into MySQL, tests validate structure, then the backend API serves them live
instead of exporting to JSON. `make export` either goes away or becomes a
cache-warming step.

## 5. Hosting recommendation (cost/complexity)

Hostinger's regular hosting plan is FTP/static-file oriented — fine for the
current stateless games, not a good fit for a persistent Python API + auth +
sessions. Two realistic paths:

| Option | Cost | Complexity | Notes |
|---|---|---|---|
| **A. Split hosting** — static frontend stays on Hostinger subdomain; backend API + MySQL move to **Railway or Render** | ~$5–10/mo on Railway/Render hobby tier (MySQL add-on included or a few $ more) | Low-medium — one new deploy target, CORS config between the two domains | **Recommended.** Keeps your sunk cost in the Hostinger subdomain/DNS, isolates the new complexity (auth, sessions, DB) to a platform built for it |
| **B. Full move** — frontend + backend + DB all on Railway/Render/Fly, drop Hostinger subdomain entirely | Similar or slightly higher | Low — one deploy target, one place to debug | Cleaner long-term, but throws away the FTP/Git setup you already have working |

**Recommendation: Option A to start.** It's the smallest delta from your
current DEPLOY.md setup — you keep the subdomain and DNS as-is, and only add
a new deploy target (Railway/Render) for the parts that are genuinely new
(auth, live API, per-user data). If the project grows and Hostinger's static
hosting starts feeling like unnecessary indirection, migrating fully later
(Option B) is a small step from there.

Either way, `.github/workflows/deploy.yml` gets a second job: deploy backend
to Railway/Render (via their CLI or Docker image push) alongside the existing
FTP-to-Hostinger job for the frontend.

## 6. Auth (email/password)

- Backend-owned: `users` table with salted+hashed passwords (e.g. `argon2` or
  `bcrypt` via `passlib`), email verification optional for v1.
- Session via short-lived JWT or server-side session cookie — JWT is simpler
  to reason about with a decoupled frontend/backend on two domains, but
  requires care with storage (httpOnly cookie, not localStorage, to avoid
  XSS token theft).
- No need to build password reset infra on day one if you're the only content
  admin and early users are a small beta group — but plan the `users` table
  with a `password_reset_token` column so it's a small add later, not a
  migration headache.

## 7. Games: the Python question

You mentioned wanting to author custom games in Python — including
non-trivial ones like a platformer. This is worth being explicit about,
because **browsers don't run Python natively**, so "games in Python" splits
into two genuinely different strategies:

### Option 1 — Python as content/logic generator, JS/HTML5 as runtime
You write game *definitions* in Python (question banks, matching pairs, even
level layouts for a platformer) as data, and a JS/Canvas (or a small game
framework like Phaser/Kaboom.js) renders and runs it in-browser. This is a
direct extension of what the current pipeline already does (Python produces
JSON, frontend consumes it) — just applied to richer game types, not only
flashcards/quiz/matching.
- **Pros:** proven pattern here already, fast, no exotic runtime, small
  bundle sizes, works everywhere including low-end phones.
- **Cons:** actual game *behavior* (physics, platformer collision, etc.) has
  to be written in JS, not Python — Python's role stays "author/config,"
  not "runtime."

### Option 2 — Real Python at runtime via Pyodide (WASM)
Ship actual Python game code to the browser and run it via Pyodide
(CPython compiled to WebAssembly), optionally with `pygame-ce` +
`pygbag` for actual `pygame` platformer code running client-side.
- **Pros:** you write real gameplay logic in Python, including physics/movement
  for a platformer, and it just runs in the browser.
- **Cons:** meaningfully heavier (multi-MB WASM runtime download, slower cold
  start), rougher edges on mobile Safari, smaller ecosystem/community support
  than mainstream JS game frameworks, harder to debug in-browser.

**Recommendation:** start with **Option 1** for the near-term thematic
games (flashcards/quiz/matching/sentence-builder — all straightforward as
JS + Python-authored data), and treat **Option 2 (Pyodide/pygbag)** as an
explicit, opt-in track for the platformer-style stretch goal later, once the
core lesson+track+progress system is live. Don't block v1 on solving the
Python-in-browser problem.

## 8. Roadmap (phased)

**Phase 0 — Foundation**
- Design DB schema: tracks, stages, lessons, vocab, grammar, games, users, progress
- Stand up FastAPI backend + MySQL on Railway/Render
- Basic email/password auth (signup, login, session)

**Phase 1 — Content + core games**
- Author 1–2 full tracks manually (e.g. "Survival Indonesian" + "Culture Basics")
- Port existing flashcards/quiz/word-matching games to read from the live API
  instead of static JSON, gated behind login for progress tracking
- Progress rollup UI (per track/stage completion)

**Phase 2 — Frontend polish + more tracks**
- Track/stage browsing UI, dashboard, streaks/basic gamification (optional)
- Expand content library across more thematic tracks

**Phase 3 — Custom game types**
- New JS-rendered game types (sentence builder, listening comprehension)
- Prototype Pyodide/pygbag platformer as a stretch feature, isolated from core
  lesson flow so it doesn't block anything else

## 9. Open questions to resolve before build starts

- Do you want a free tier + paid tier, or fully free for now?
- Any target number of tracks/lessons for a "launch-ready" v1 (so scope has a
  concrete stopping point)?
- Should progress be shareable/social (leaderboards, streak sharing) or
  strictly private?
- Content language for lesson explanations — English only, or bilingual
  EN/ID toggle?
