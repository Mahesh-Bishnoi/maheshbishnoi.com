# Plan 01 — maheshbishnoi.com Stable Portfolio / Blog / Resume

## Goal
Serve `maheshbishnoi.com` + `www.maheshbishnoi.com` from Cloudflare Workers Static Assets. No origin server. Free SSL, cache, analytics.

## Current state (verified 2026-10-15)
- Zone `maheshbishnoi.com` id `15808edc1a901c6c400ee3552c7e1b91`, status `active`, plan Free.
- DNS: `www` CNAME -> `maheshbishnoi.com` (proxied), no apex A/AAAA, so apex does not resolve. TXT: `v=spf1 -all`, `_dmarc`, `*._domainkey` placeholder.
- Account: `2a8c9c88f4c1b28de68ba60b753acba5` (`Bishnoimahesh96@gmail.com's Account`), email `bishnoimahesh96@gmail.com`.
- Repo `/home/mahesh/code/cf`: `package.json` has `cf@^1.0.0-beta.17`, no `wrangler.*`, no `cloudflare.config.ts` -> use `cf` per AGENTS.md.
- Auth: `./node_modules/.bin/cf auth whoami` OK (OAuth token).

## Architecture
- Workers project in this repo `/home/mahesh/code/maheshbishnoi.com/` with Astro static output `dist/` (`public/` for favicon, robots.txt). Deploy via Wrangler (`wrangler.jsonc` assets-only, no `main`) — interim until `cf` supports Astro 6+; `cf` used for DNS/zone management. Future: `cf migrate` to `cloudflare.config.ts` (`cloudflare.config.ts.cf-future` holds draft).
- Custom domains: apex + `www` attached to Worker. Apex uses CNAME flattening / proxied records — no manual A records.
- Email preserved: keep existing SPF/DMARC/DKIM TXT untouched.
- Free services: Workers Static Assets, SSL/TLS (Full), Cache Rules, Web Analytics, Email Routing (existing).

## Steps
1. Scaffold Astro (static output) in this repo: DONE — `output: 'static'`, `dist/` with `/`, `/blog/`, `/resume/`, `/404.html`.
2. Local dev: `npm run dev` / `npm run build`. Verify `dist/` emits 4 pages.
3. Deploy via Wrangler (interim, Astro 7 works):
   ```bash
   npm run build
   ./node_modules/.bin/wrangler deploy --dry-run
   ./node_modules/.bin/wrangler login   # one-time browser auth
   npm run deploy
   ```
   `cf` does not build Astro 6+ during beta (missing Build Output spec), so Wrangler owns deploys for now.
4. Attach domains: apex `maheshbishnoi.com` + `www.maheshbishnoi.com` via Workers custom domains (wrangler or dashboard). Verify DNS:
   ```bash
   ./node_modules/.bin/cf dns records list --zone maheshbishnoi.com
   dig +short maheshbishnoi.com
   dig +short www.maheshbishnoi.com
   curl -sI https://maheshbishnoi.com | head
   curl -sI https://www.maheshbishnoi.com | head
   ```
6. Harden: SSL/TLS Full (Strict works with Workers), Single Redirect Rule `www.maheshbishnoi.com` -> apex `maheshbishnoi.com` 301, Cache Rule for `/*.(css|js|png|jpg|webp|woff2)`, enable Web Analytics.
7. Email check: ensure TXT records unchanged, test Email Routing if enabled.

## Verification
- `cf zones get --zone maheshbishnoi.com` shows active.
- `https://maheshbishnoi.com` 200, valid cert, `https://www.maheshbishnoi.com` 301 -> apex.
- No change to `_dmarc`, `spf`, `_domainkey` records.

## Rollback
- `wrangler versions list` / rollback to previous version, or remove custom domains to restore prior `www` CNAME only. DNS export saved at `/tmp/opencode/dns-export.txt`:
  ```bash
  ./node_modules/.bin/cf dns records export --zone maheshbishnoi.com
  ```

## Future: migrate to `cf`
- When `cf` supports Astro 6+ builds (or site moves to plain Vite), run `cf migrate` to convert `wrangler.jsonc` -> `cloudflare.config.ts`. Draft already in `cloudflare.config.ts.cf-future`. Then `cf deploy --dry-run` must emit `.cloudflare/output/v0/config.json` before real deploy.

## Resolved decisions
- Location: this repo `maheshbishnoi.com` holds all code (plan copied here from `cf` folder).
- Stack: Astro from start, `output: 'static'`, `dist/` deployed to Workers Static Assets.
- Canonical: apex `maheshbishnoi.com` serves 200, `www` 301 -> apex. Apex CNAME note: DNS spec forbids CNAME at apex (conflicts with SOA/NS); Cloudflare solves via CNAME flattening + Workers custom-domain managed records, so no manual A records needed. Portability risk only if DNS moves off Cloudflare.
