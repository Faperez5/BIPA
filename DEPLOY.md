# Deployment

## Architecture

The main site is built with Hostinger's **AI Website Builder** — a no-code
product with no File Manager, FTP/SSH, or Git integration, so it can't host
this repo's custom static games or be driven by CI/CD.

Instead, the BIPA games (flashcards, quiz, word-matching) are hosted on a
**subdomain** (e.g. `games.yourdomain.com`) under the regular Hostinger
Web/Cloud/Business hosting plan, which does support FTP and hPanel's Git
integration. The AI-builder site links or embeds to that subdomain.

hPanel's own Git auto-deploy (Advanced → Git) is *not* used for the games
subdomain: it only pulls raw files as-is and can't run the Python/MySQL
content pipeline that produces `public/data/*.json`. `.github/workflows/deploy.yml`
runs that pipeline (`make migrate`, `make load`, `make test`, `make export`)
and then FTPs the resulting `public/` directory to Hostinger.

## One-time Hostinger setup (manual, in hPanel)

1. Under the regular hosting plan, create a subdomain — e.g. `games.yourdomain.com` —
   and note its document root path.
2. Create an FTP/SFTP user scoped to that document root only.
3. In GitHub, add these repo secrets (Settings → Secrets and variables → Actions):
   - `HOSTINGER_FTP_SERVER`
   - `HOSTINGER_FTP_USERNAME`
   - `HOSTINGER_FTP_PASSWORD`
   - `HOSTINGER_FTP_PATH` — the document root from step 1
4. In the AI Website Builder, add a nav link or an "Embed code" block
   (iframe or plain link) pointing at `games.yourdomain.com`.

## CI/CD

Every push to `main` triggers `.github/workflows/deploy.yml`, which builds the
content pipeline against a throwaway MySQL service container and deploys
`public/` to the games subdomain. No manual upload needed after the one-time
setup above.
