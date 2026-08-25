# Accounts, Costs & Signups

*Everything you personally need to obtain, in the order you'll need it.
Prices checked 2026-08-23 — verify before paying, they move.*

**Bottom line:** beyond the Hostinger plan you already have, this project needs
**one new paid account (Railway, ~$5/month)** and everything else has a free
tier that fits comfortably. Realistic running cost is **$5–15/month**.

---

## Already have — worth verifying

- [ ] **Domain name.** Confirm the registrar and expiry date, and that
      auto-renew is on. Everything below assumes you control DNS for it.
- [ ] **Hostinger hosting plan.** You need the *regular* Web/Cloud/Business
      plan, not just the AI Website Builder — the builder has no FTP and can't
      host the app. Check that your plan allows at least 2 subdomains and 1 FTP
      account; entry-level plans cap exactly there, which is just enough.
- [ ] **Check your Hostinger renewal date and price.** This is the one that
      bites people: introductory rates around $2–3/month apply only on long
      terms, and renewals run roughly $11–12/month. Know which number you're on
      and when it changes, so it isn't a surprise mid-project.

---

## Before you write any code — tasks 1–4

| Need | Cost | Card? | Email? |
|---|---|---|---|
| **GitHub account** | Free | No | Yes |
| **Docker Engine** | Free | No | No |
| **Python 3.13, uv, Node 24** | Free | No | No |

- **GitHub free plan** gives unlimited private repos, 2,000 Actions minutes per
  month on private repos, and 500 MB of artifact storage.
  **Consider making the repo public.** Actions minutes are unlimited and free on
  public repositories, so it removes your only real CI budget constraint — and
  your README already frames this as a DevOps practice project, which is
  portfolio material. The tradeoff is that your course content becomes readable
  by anyone.
- **Docker** — you're on Linux, so install Docker Engine from your package
  manager. It's fully free with no licensing question. (Docker Desktop's
  commercial-use terms only matter on macOS and Windows.)

---

## To go live — task 17

| Need | Cost | Card? | Email? |
|---|---|---|---|
| **Railway account** | Trial: one-time $5 credit, expires in 30 days. Hobby: $5/month | Likely yes for Hobby | Yes — sign in with GitHub |

- Railway's Hobby plan is **$5/month and includes $5 of usage credit**, with
  CPU, memory, volumes, and egress billed on top of that. For a low-traffic
  Django app plus a small Postgres, expect to sit at or near the $5 floor.
- **Postgres is not a separate purchase** — it's a Railway service that draws
  from the same usage budget.
- Sources conflict on whether the trial requires a card. Assume you'll need one
  for the Hobby plan; the trial may not.
- **The 30-day trial expiry matters for your schedule.** Don't start the trial
  until you're actually at task 17 — burning it during Phase 0, when nothing is
  deployable yet, wastes it entirely.

---

## When you add art and audio — task 38

| Need | Cost | Notes |
|---|---|---|
| **Git LFS storage** | Free: 1 GB storage + 1 GB/month bandwidth. Paid data packs beyond | Verify current pack pricing before you exceed it |
| **Tiled** map editor | Free (donationware) | Already in the pinned stack |
| **Sprite editor** | Aseprite ~$20 one-time, or free: Krita, LibreSprite, Piskel | Aseprite is the standard for pixel art |
| **Art assets** | Free options exist | Kenney.nl is CC0 and free; itch.io has both free and paid packs |
| **Audio** | Free options exist | freesound.org, plus CC0 packs on itch.io |

1 GB of LFS is a lot of 2D sprite work — you're unlikely to hit it, but the
bandwidth limit is the one that surprises people, since every CI run that
checks out assets consumes it.

---

## Optional, only when you need them

| Need | When | Cost |
|---|---|---|
| **Transactional email** (Resend, Postmark, Brevo) | Only when you add password reset — task 14 deliberately skips it | Free tiers cover early usage |
| **Error tracking** (Sentry) | Once real users exist | Free tier is generous for solo projects |
| **Uptime monitoring** (UptimeRobot) | Once you'd care about downtime | Free tier sufficient |

None of these are needed to launch. Don't sign up preemptively.

---

## Cost summary

| Stage | Monthly |
|---|---|
| Phases 0–1 (nothing deployed) | **$0** on top of existing Hostinger |
| From task 17 (API live) | **~$5** (Railway) |
| Steady state with content and traffic | **$5–15** |

One-time: ~$20 if you buy Aseprite, otherwise $0.

---

## Where credentials and account details live

**Nothing sensitive goes in this repository.** Git history is permanent, CI
logs can echo values, and forks carry everything — and if you take the
public-repo suggestion above, "private" stops being a safety net at all.

| Kind of thing | Where it lives |
|---|---|
| Passwords, recovery codes, 2FA seeds | **Password manager** — Bitwarden (free), KeePassXC (free), 1Password (paid) |
| Deploy tokens, API keys used by CI | **GitHub Actions secrets** — the six listed in DEPLOY.md Part 3 |
| Runtime config and secrets in production | **Railway environment variables** |
| Local development values | **`.env`**, gitignored. `.env.example` is committed and lists the *names* only |
| Non-secret identifiers — which plan, which region, document root path | **`_docs/accounts.local.md`**, gitignored (template below) |

The pattern is that the repo records *what* exists and *where the value is
kept*, never the value itself. Anyone — you in a year, or an agent — can then
find their way to the right place without the secret ever being in git.

Turn on 2FA for GitHub, Hostinger, and Railway, and store the recovery codes in
the password manager. Losing GitHub access means losing the deploy pipeline.

### Local notes template

`_docs/accounts.local.md` is gitignored. Keep identifiers and paths there —
still no passwords, since those belong in the password manager:

```markdown
# Local account notes — NOT COMMITTED

## Domain
Registrar:            Expiry:            Auto-renew:

## Hostinger
Login email:          Plan tier:         Renewal date / price:
Subdomain:            Document root:
FTP username:                            (password → password manager)

## Railway
Login:  GitHub SSO    Project name:      Region:
Custom domain:  api.<domain>             (token → GH secret RAILWAY_TOKEN)

## GitHub
Account:              Repo:              Public or private:
```

---

## Signup order

Work through these only as you reach them — several have clocks that start
ticking on signup.

1. **Now:** GitHub account, Docker Engine, Python/uv/Node toolchain.
2. **Verify now, act if needed:** Hostinger plan tier, subdomain allowance,
   renewal date.
3. **At task 17:** Railway. Not before — the trial credit expires in 30 days.
4. **At task 38:** art and audio tooling, only if the free options don't suit.
5. **After launch:** email sending, error tracking, monitoring.
