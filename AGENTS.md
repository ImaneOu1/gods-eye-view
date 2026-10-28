# AGENTS.md — God's Eye View in the Base44 sandbox

Notes a fresh session would otherwise have to rediscover. The repo's own
`README.md` / `CONTRIBUTING.md` remain the reference for the app itself.

## Running it here

```bash
docker compose -f docker-compose.base44.yml up -d        # high timeout
docker compose -f docker-compose.base44.yml logs -f web
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/    # expect 200
```

- One service, `web`: the Vite dev server **is** the app. It serves the client
  and runs every API proxy (ADS-B, AIS, Overpass, OpenAI Realtime, …) as
  middleware, so there is no separate backend, no database and no migrations.
- Runs from the bind-mounted checkout on a plain `node:24.14.0-slim` image
  (same digest the repo's Dockerfile pins) — never the repo's own image, which
  `COPY`s the source and would freeze edits.
- Host port **3000** maps to container port 3000 (`HOST=0.0.0.0 PORT=3000`) so
  Vite's dev client opens its HMR socket on the page's own address through the
  preview proxy. Do not move the internal port without re-checking HMR.
- First start installs with `npm ci` into the checkout's `node_modules` and
  stamps `node_modules/.base44-lock`. Later starts skip it; a changed
  `package-lock.json` triggers a reinstall. To force one, remove the stamp.
  `PUPPETEER_SKIP_DOWNLOAD=1` avoids the ~150 MB Chromium download (QA scripts
  only).
- `node_modules` lives in the repo (gitignored), not an anonymous volume, so it
  survives container recreation.

## Sandbox-specific behaviour

- **Host allowlist.** `build/allowedHosts.js` ignores suffix/wildcard entries by
  default. `BASE44_PREVIEW_MODE=1` (passed through by compose) makes
  `server/standalone/vite.config.js` opt in, so `GEV_ALLOWED_HOSTS=.${BASE44_SANDBOX_HOST_DOMAIN}`
  is honoured while the preview proxy delivers the app as
  `<port>-<sandbox id>.e2b.app` — that id rotates, so the exact name cannot be
  pinned. Unset, the strict "explicit hostnames only" policy is unchanged and
  the covering tests still pass.
- Nothing else in the app is overridden for the sandbox: the app's CSP
  (`frame-ancestors 'none'` + `X-Frame-Options: DENY`) is left intact; the
  preview proxy handles the iframe embedding.
- Provider Settings (the in-app POWER UP panel) trusts loopback plus the
  container gateway, which `scripts/docker-start.mjs` discovers at start. Keys
  typed there are written to the checkout's `.env`. The Base44 dashboard route
  is the compose `env_file: /run/base44/app.env` (outside the repo) — that file
  is the *last* entry, and no user-supplied key may be added under
  `environment:`, which would outrank it.

## Credentials

The app runs **keyless**: with no key at all it starts on keyless Esri World
Imagery with OSM in the map tray. Every credential in `.env.example` is
optional and only fills in one layer or provider (see `Keys & Costs` in the
README). Missing keys are therefore never a boot failure — check
`.base44/environment.json` for the declared list and the preview logs before
assuming otherwise.

## Verifying

- Tests run inside the container (the sandbox host has no Node):
  `docker compose -f docker-compose.base44.yml exec -T web node --test src/tooling/allowedHosts.test.mjs`
  for one file, `… exec -T web npm test` for the full suite (needs `git` in the
  image — the runtime image has none, so install it first or run the suite in
  the repo's own Dockerfile image).
- Host policy check, worth repeating after any allowlist change:
  `curl -s -o /dev/null -w '%{http_code}' -H 'Host: evil.example' http://localhost:3000/`
  must print `403`.
- The page is live source, not a build: `/@vite/client` must answer 200 and
  `index.html` must reference it.
