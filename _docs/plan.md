# Belajar Bahasa — Platform Plan
*Indonesian-for-expats learning site: a turn-based roguelike, lessons, and culture, built on top of the existing games-subdomain infrastructure*

## 1. Vision

Turn the current BIPA games subdomain into a self-serve Bahasa Indonesia learning
platform for expats and culturally-curious English speakers. Learners pick
**thematic tracks** (Survival, Work, Culture, Travel, etc.), each with an internal
difficulty progression, mixing **structured lessons** and **games**, with an
**account** that tracks progress across both.

The centrepiece game is a **turn-based roguelike** whose encounters are resolved
through Indonesian dialogue and language challenges — the language is the
mechanic, not a wrapper around one. See §7.

This is a step up from the current architecture: today's pipeline produces
static JSON consumed by stateless games. Accounts, progress, and server-owned
run state require a live backend and database, not a nightly export.

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
| **VocabItem / GrammarPoint** | Shared atomic units, reusable across lessons, dialogue, and encounters |
| **DialogueNode** | One beat of NPC speech, FK to the VocabItems/GrammarPoints it exercises |
| **DialogueChoice** | A player reply option on a node — the Indonesian text, its correctness/consequence, and the next node |
| **Encounter** | A roguelike encounter template: entry dialogue, win/lose consequences, vocab pool it draws from |
| **Run** | One roguelike playthrough — seed, current state, meta-progression carried forward |
| **UserProgress** | Per-user completion + score per lesson/encounter, rollup per track/stage |

**Dialogue is content, not code.** Dialogue trees are the primary lesson
delivery mechanism, so they live in the database with real foreign keys to
VocabItem/GrammarPoint — never hardcoded in the game. That makes them
authorable in the admin UI and validatable in CI (see §5).

Because you're authoring content manually, the schema should optimize for
**your** editing speed (good FK reuse of VocabItem/GrammarPoint across tracks
and dialogue) more than for automated ingestion.

---

## 4. Tech stack (decided)

| Layer | Choice | Why |
|---|---|---|
| Backend | **Django** (Python) | Ships auth, sessions, migrations, ORM, and — critically — the **Django admin as a free CMS** over tracks/lessons/vocab/dialogue. Solves the authoring problem the plan otherwise leaves unbudgeted. |
| Database | **Postgres** | Run state and dialogue trees are nested JSON; JSONB indexing is the right tool and MySQL's JSON support is weaker. Nothing in the schema needs MySQL. Also the first-class managed DB on every candidate host. |
| Game rules | **Python, server-side** (Django) | Turn-based means no latency budget, so encounter resolution, run state, RNG seeds, and answer-checking all live server-side, next to the content DB. Also makes progress unforgeable. |
| Frontend | **React 19 + Vite 7** (TypeScript) | Static SPA — required, since the Hostinger subdomain serves files over FTP and can't run a server-rendered app. |
| Game presentation | **Phaser 4** (TypeScript) | Canvas for the map/encounter view. Same JS context as the page, so it shares the session cookie and calls the API directly. |
| Text UI | **React DOM** over the canvas | Dialogue, choices, glosses, inventory, "add to review deck". A turn-based dialogue game is mostly UI — this is the majority of the interface. |
| Hosting | **Hostinger + Railway** (split) | Hostinger keeps the marketing site and the games subdomain exactly as today; Railway adds only the persistent Python process Hostinger can't run. |
| Assets | Git LFS in-repo → FTP'd to Hostinger with the bundle | Same origin as the page that loads them: no CORS, no separate CDN. |

### Alternatives considered

- **FastAPI + React** — most from-scratch learning value, but you hand-build
  auth, migrations, and the entire admin/CMS. Rejected: too slow to content.
- **Godot 4** — better engine, but its web export is a sealed canvas: no shared
  DOM, so every gloss/tooltip/review-capture overlay gets rebuilt inside it, and
  auth needs an explicit token handoff. That integration tax is paid on every
  feature. Turn-based also removes the animation/action tooling that was its
  main advantage.
- **Pyodide / pygbag** (real Python in-browser) — dropped entirely. Server-side
  Python gives the same "gameplay logic in Python" benefit without the multi-MB
  WASM download, mobile Safari fragility, or in-browser debugging pain.
- **Supabase / PocketBase** — fastest to live, but deletes the DevOps practice
  the repo exists for, and can't host server-authoritative game rules.
- **Next.js / Laravel** — both abandon Python for the backend.

### Shape

```
yourdomain.com          Hostinger AI Builder — marketing site (unchanged)
       │ links to
games.yourdomain.com    Hostinger FTP — static SPA + game assets (existing deploy job)
       │ fetch(), credentials: include
api.yourdomain.com      Railway — Django: auth, content API, admin CMS,
       │                          and server-authoritative game rules
Railway Postgres        content + dialogue + users + runs + progress
```

Both subdomains sit under the same registrable domain, so the session cookie
scoped to `.yourdomain.com` is **first-party** and survives third-party cookie
blocking. Full topology, settings, and pinned versions: **[ARCHITECTURE.md](ARCHITECTURE.md)**.

---

## 5. What changes vs. the current DEPLOY.md setup

### Stays
- The CI concept — validate content, run tests, deploy on merge to `main`.
- Content validation as a first-class CI step, now more valuable than before:
  tests catch broken cross-references between lessons, dialogue, and vocab
  (e.g. *"this dialogue choice uses a VocabItem the player hasn't been taught
  in this track"*).
- Hostinger AI Website Builder as the marketing/landing site.

- **The FTP-to-Hostinger deploy job.** It retargets from `public/` to Vite's
  `dist/`, but the mechanism, the subdomain, and the four FTP secrets are
  unchanged. See DEPLOY.md, which remains accurate.

### Goes away
- **Static JSON export.** Accounts, progress, and run state are read/write per
  user at request time. `make export` either disappears or becomes a
  cache-warming step.
- **MySQL.** Replaced by Postgres.

### New
- **A second deploy target.** `.github/workflows/deploy.yml` gains a
  `deploy-api` job (Docker image → Railway → `migrate`) alongside the existing
  FTP job, both gated behind a shared `test` job.
- **Cross-origin config** — CORS, CSRF trusted origins, and a cookie domain of
  `.yourdomain.com`. The price of keeping the split; see ARCHITECTURE.md §1.
- **Asset pipeline** — Git LFS for source art/audio, shipped to Hostinger with
  the frontend bundle.
- **Migrations in CI** against a Postgres service container.

---

## 6. Auth (email/password)

Django's built-in auth covers nearly all of this out of the box — hashing
(PBKDF2 by default, Argon2 available), sessions, password reset, and the
permission system for admin access. Same-origin session cookies mean no CORS
config and no JWT-in-localStorage XSS question.

Remaining decisions: whether to require email verification for v1 (probably
not, for a small beta) and whether to add social login later (Django-allauth
if so).

---

## 7. The game

**Decided: turn-based roguelike.** Runs, permadeath, meta-progression between
runs, randomized encounters — Slay the Spire's structure with Hades' dialogue
and relationship depth. Encounters resolve through dialogue choices and
language challenges rather than real-time combat.

### Why turn-based

- **No frame budget**, so a server round-trip per turn is fine — which is what
  lets game rules live in Python (§4). With real-time action, Python could only
  ever have been a config generator.
- **Works on mobile**, and gives the learner time to actually read Indonesian
  rather than reacting to it.
- **The language is the mechanic.** Choosing the right register when speaking
  to an elder, or the right word for a market negotiation, *is* the encounter.
- **Solo-feasible.** A real-time action roguelike at Hades' fidelity is a studio
  product; a farming sim like Stardew was years of full-time solo work. Neither
  is compatible with also authoring a language curriculum.

### Division of responsibility

| Concern | Where it lives |
|---|---|
| Encounter resolution, run state, RNG seed, scoring, answer-checking | Django (Python), server-authoritative |
| Map view, sprites, transitions, encounter staging | Phaser 4 (canvas) |
| Dialogue text, choices, glosses, inventory, review-deck capture | React DOM over the canvas |
| Dialogue trees, vocab, encounter templates | Postgres, edited via Django admin |

Consider **Ink** (`inkjs`) as the dialogue authoring format if branching logic
outgrows a plain node/choice table — but author it against the vocab tables
either way, so CI can still validate cross-references.

---

## 8. Roadmap (phased)

**Phase 0 — Foundation**
- Design the Postgres schema: tracks, stages, lessons, vocab, grammar,
  dialogue nodes/choices, encounters, runs, users, progress
- Stand up Django + Postgres on Railway; point `api.yourdomain.com` at it;
  Docker Compose for local dev
- Django auth (signup, login, session) + admin configured for content authoring
- CI: migrations + tests against a Postgres service container

**Phase 1 — Content + vertical slice**
- Author 1 full track manually (e.g. "Survival Indonesian") including dialogue
- One playable encounter end to end: DOM dialogue UI → server resolution →
  progress recorded. No Phaser yet — prove the loop in plain HTML first.
- Content validation tests (vocab cross-reference checking)

**Phase 2 — The roguelike loop**
- Run structure: seed, encounter sequence, permadeath, meta-progression
- Phaser 4 map/encounter view layered under the existing React DOM UI
- Asset pipeline (Git LFS → FTP to Hostinger)
- Progress rollup UI (per track/stage completion)

**Phase 3 — Breadth**
- Expand content across more thematic tracks
- Supporting game types (flashcards, quiz, sentence builder) as lesson practice
- Dashboard, streaks / basic gamification (optional)

---

## 9. Open questions to resolve before build starts

- Do you want a free tier + paid tier, or fully free for now?
- Any target number of tracks/lessons for a "launch-ready" v1 (so scope has a
  concrete stopping point)?
- Should progress be shareable/social (leaderboards, streak sharing) or
  strictly private?
- Content language for lesson explanations — English only, or bilingual
  EN/ID toggle?
- Art direction and asset sourcing — commissioned, asset-store, or public
  domain? This gates Phase 2 and is easy to underestimate.
