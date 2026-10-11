# Deploying NexoraTools to Vercel

Vercel hosts only `apps/web` (Next.js). The Fastify API, BullMQ worker, Postgres, Redis and the shared
file storage need long-running processes, Ghostscript/poppler binaries and a persistent disk, so they
must run elsewhere (VPS, Railway, Fly.io, Render...) using `docker-compose.yml` minus `web`/`nginx`.

## 1. Host the API + worker (Docker)
On a server with Docker, ports 80/443 open, and a DNS record (e.g. `api.example.com`) pointing at it:
```bash
ENV_TEMPLATE=.env.backend.example sh scripts/init-env.sh you@example.com   # .env with generated secrets + admin
# edit .env: API_DOMAIN=api.example.com, WEB_ORIGIN=https://<your-vercel-domain>
docker compose -f docker-compose.backend.yml up -d --build
curl https://api.example.com/api/health
```
`docker-compose.backend.yml` runs postgres, redis, migrate, api, worker and Caddy (automatic HTTPS, only `/api/*` is proxied).
`WEB_ORIGIN` must equal the exact origin users open; the API rejects other Origins. Cookies are `Secure` by default.

## 2. Create the Vercel project
1. Import the repo. Set **Root Directory** to `apps/web` (keep "Include source files outside of the Root Directory" enabled).
2. Framework preset: Next.js. `apps/web/vercel.json` already sets the install/build commands.
3. Add env var `API_INTERNAL_URL=https://api.example.com` (Production, and Preview if wanted). The build fails if it is missing.
4. Deploy.

## Notes
- The browser stays same-origin: Vercel rewrites `/api/*` to your API, so cookies and CORS work unchanged.
- Preview deployment URLs differ from `WEB_ORIGIN`, so login/uploads will be blocked there unless you add that origin.
- Vercel Functions have a 4.5 MB request-body limit. Uploads here go through an external rewrite, not a Function,
  but this is untested: try a 20-50 MB upload first. If it fails, serve the API on a same-site subdomain and upload directly.
- Caddy trusts `X-Forwarded-For` from any source (Vercel's IPs are not a fixed range) so rate limits see real client IPs.
  Someone hitting the API domain directly could spoof it; Vercel itself overwrites that header.
- Untested: Vercel must send a Host header Caddy accepts. If requests return empty/404 through Vercel but work by
  direct curl, check that first.
