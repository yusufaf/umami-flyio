# Umami on Fly.io

Fly.io deployment config for [Umami](https://github.com/umami-software/umami), a
cookieless, GDPR-friendly, self-hosted alternative to Google Analytics.

Umami ships an official `docker-compose.yml` with two services. This maps them onto
**two Fly apps** in one org, connected over Fly's private network:

| Fly app | image | exposure |
|---|---|---|
| `umami-db-af` | `postgres:18.3-alpine` | private (`.internal`), always-on |
| `umami-af` | `ghcr.io/umami-software/umami:postgresql-latest` | **public** HTTPS, auto-suspend |

Umami v3 dropped MySQL support — Postgres only. The app is stateless (all data lives
in Postgres), so only `db` has a volume.

## Why self-host instead of Umami Cloud, and why the renamed script

Umami Cloud's free tier caps out at 100k events/mo, 3 websites, 6 months retention —
tight if you're tracking more than a couple of sites. Self-hosting removes all three
limits for the cost of a $3-ish/mo Fly app.

**Renaming the tracker matters more than self-hosting does.** EasyPrivacy (the
filter list behind uBlock Origin's default block) matches Umami by *path*, not just by
host — `/umami.js`, `` ||umami.*/script.js|$script ``. A self-hosted instance serving
the default `/script.js` from `analytics.example.com` gets blocked exactly like
`cloud.umami.is` does. Two things fix it, both already wired into `app/fly.toml`:

1. **Serve from a subdomain of the site you're tracking** (e.g.
   `analytics.yourdomain.com` tracking `yourdomain.com`) — same registrable domain, so
   ad blockers treat it as first-party and skip every `$third-party` rule.
2. **Rename the script and the collect endpoint** off their defaults via
   `TRACKER_SCRIPT_NAME` / `COLLECT_API_ENDPOINT` — defeats the generic path-based
   rules. This repo sets them to `s.js` and `/api/e`; pick your own if you'd rather.

If a site can proxy same-origin (Netlify redirects, a Vite dev-server proxy, etc.),
that's stronger still — see "Same-origin proxying" below.

## ⚠️ Choose your own app names first

**Fly.io app names are globally unique** (shared across all Fly users), so the example
names below — `umami-db-af`, `umami-af` — are almost certainly already taken. Pick your
own (e.g. add a personal suffix: `umami-db-af-xyz`).

If you change a name, update it in **every** place it appears:

| What | Where |
|---|---|
| db app name | `db/fly.toml` → `app` |
| app app name | `app/fly.toml` → `app` |
| db hostname (app→db) | `DATABASE_URL` secret (`<db-app>.internal`) |

Rename all at once (run in repo root, swap in your names):
```bash
sed -i 's/umami-db-af/YOUR-db/g; s/umami-af/YOUR-app/g' db/fly.toml app/fly.toml README.md
```
Then use the same names in the `fly` commands below.

## Prerequisites
- [`fly` CLI](https://fly.io/docs/flyctl/install/) installed and `fly auth login` done
- `openssl` (for generating secrets)

## Cost (rough, LA region)
db 256mb always-on ≈ $2.02/mo + 1GB volume ≈ $0.15/mo + app machine, mostly suspended
(auto-stop) ≈ $0.50–1/mo ≈ **~$2.50–3.50/mo** total. (shared-cpu-1x: 256mb $2.02, 512mb
$3.32; volumes $0.15/GB/mo.) A custom domain and its TLS cert are free (shared IPv4
included; first 10 certs per org are $0).

New Fly accounts have **no free tier** — this is pay-as-you-go.

**Don't use Fly Managed Postgres** for this — its cheapest plan is $38/mo, ~15x what
the whole rest of this setup costs. If you'd rather not run the `db` app at all,
[Neon's free tier](https://neon.com) (0.5GB, scale-to-zero) works as a drop-in
`DATABASE_URL` — drop `db/fly.toml` and point the app secret at Neon instead.

## Deploy

Generate secrets first and keep them handy (don't commit them):
```bash
DBPW=$(openssl rand -hex 16)
APPSECRET=$(openssl rand -hex 32)
```

### 1. Database
```bash
fly apps create umami-db-af
fly volumes create umami_pgdata --app umami-db-af --region lax --size 1
fly secrets set --app umami-db-af \
  POSTGRES_DB=umami POSTGRES_USER=umami POSTGRES_PASSWORD="$DBPW"
fly deploy --config db/fly.toml
fly logs --app umami-db-af   # watch for: "database system is ready to accept connections"
```

### 2. App (public)
```bash
fly apps create umami-af
fly secrets set --app umami-af \
  APP_SECRET="$APPSECRET" \
  DATABASE_URL="postgresql://umami:$DBPW@umami-db-af.internal:5432/umami"
fly deploy --config app/fly.toml
fly logs --app umami-af   # watch for: "Umami version X running" — no DB connection errors
```

Open `https://umami-af.fly.dev`.

## First login

Umami's first boot creates an `admin` account with password `umami`. **This instance
is public the moment the cert resolves** — log in and change the admin password
immediately, before adding any website. Settings → Profile → Change password.

Then: **Settings → Websites → Add website**, enter the domain you're tracking, and
copy the `data-website-id` UUID it generates — that's what goes in the tracker
`<script>` tag on the site itself, e.g.:
```html
<script defer src="https://umami-af.fly.dev/s.js" data-website-id="YOUR-UUID"></script>
```
(Swap in your custom domain once you've added one — see below.)

## Same-origin proxying (optional, strongest ad-block resistance)

Renaming the script defeats path-based rules; serving it from the *tracked site's own
domain* additionally defeats every `$third-party` rule, since same-registrable-domain
traffic never counts as third-party. If your site's host supports request rewrites,
proxy `/s.js` and `/api/e` to this app rather than loading them cross-host. A Netlify
example, added **above** any catch-all SPA redirect (Netlify uses the first match):
```toml
[[redirects]]
  from = "/s.js"
  to = "https://umami-af.fly.dev/s.js"
  status = 200

[[redirects]]
  from = "/api/e"
  to = "https://umami-af.fly.dev/api/e"
  status = 200
```
Then point the tracker tag at the local path instead: `src="/s.js"
data-host-url="/"`. Static hosts with no rewrite support (e.g. GitHub Pages) can't do
this — a subdomain (as in "First login" above) is the next-best option and is what
this repo is set up for by default.

## Custom domain (recommended)
```bash
fly certs add analytics.yourdomain.com --app umami-af
```
Add the CNAME/A records `fly certs show analytics.yourdomain.com --app umami-af`
prints, at your DNS provider. First 10 hostname certs per org are free.

## Verify
1. `fly status` on both apps — db `started`; the app may show `suspended` until you
   hit it.
2. `fly logs --app umami-af` — no repeating DB connection errors.
3. `curl -sI https://umami-af.fly.dev/s.js` → `200`, JS content-type.
4. Load the tracked site **with an ad blocker enabled** (e.g. uBlock Origin) — Network
   tab shows `s.js` loading and the POST to `/api/e` succeeding, neither struck
   through as blocked.
5. Umami dashboard → Realtime shows the visit within seconds.
6. `fly ssh console --app umami-db-af` → `psql -U umami umami -c '\dt'` shows tables.

## Ongoing
```bash
fly logs --app <app>                 # tail logs
fly status --app <app>               # machine status
fly ssh console --app <app>          # shell in
fly deploy --config <dir>/fly.toml   # redeploy after image/config change
```
Images are pinned to `:latest` / `:postgresql-latest`; `fly deploy` re-pulls. Pin to a
digest if you want reproducibility. Umami has no built-in DB migration step to run
manually — the app runs Prisma migrations itself on boot.

## Notes / gotchas
- **`.internal`, not `.flycast`.** Postgres binds dual-stack by default, so unlike
  some images (SurrealDB, Node servers that default to IPv4-only binds), no special
  flycast address is needed here — `.internal` resolves fine over Fly's IPv6 private
  network.
- **The database is always-on by design**, same reasoning as `open-notebook-flyio`:
  auto-stopping it races the app's boot-time DB connection. At 256mb this costs
  ~$2/mo — cheap insurance against intermittent boot failures.
- **`APP_SECRET` should not change** once set — it's used to sign auth tokens;
  rotating it invalidates existing sessions (not catastrophic, just an inconvenience,
  unlike the encryption keys in other repos in this series).
- **One website id per tracked domain.** Add a website in the Umami UI per hostname
  you track (apex site, docs site, each app) — each gets its own
  `data-website-id` and its own dashboard.
- **`DISABLE_TELEMETRY = '1'`** opts this instance out of Umami's own anonymous usage
  telemetry back to the Umami team — unrelated to the visitor tracking this app exists
  to do.
