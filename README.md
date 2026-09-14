# Silent Payments Watch

Silent Payments Watch is a feed-first public-source activity tracker for BIP-352 Silent Payments and directly related work (BIP-375/376/392, scanning, wallets, hardware, and indexing).

Engine comes from [source-watch](https://github.com/macgyver13/source-watch). This repo is the instance: seeds, identity, bootstrap feed snapshot, and the Worker. To pick up template bug fixes:

```bash
git fetch source-watch
git merge source-watch/main
```

Conflicts should stay in `config/`.

This instance is **service mode**. Live site: https://silent-payments-watch.macgyver-dev.workers.dev/ — Worker + D1, not Pages.

`data/public/` is the bootstrap snapshot from the former `preview/silent-payments` static site. The first ingest uses it when D1 is empty so `discovered_at` does not reset. Later collects read live rows from D1. Public feed JSON is not committed under `site/static/`.

## Scope

This project aggregates public source metadata and activity. Inclusion is not endorsement, technical review, security assessment, production-readiness judgment, or a canonical roadmap.

## Wire the Worker (once)

Needs Node 22+, Wrangler (`npm ci`), Python 3, PyYAML, Hugo `0.164.0`, and a Cloudflare account that can finish `renderAll` (a few thousand items exceeds the free-plan 10 ms CPU cap).

1. `npm ci`
2. `npx wrangler login` if this machine is not already authenticated.
3. D1 `silent-payments-watch` already exists (`database_id` `9c742cdb-846e-4c63-a27c-028400283a92` in `wrangler.jsonc`). Do not create another.
4. Apply migrations: `npx wrangler d1 migrations apply silent-payments-watch --remote`
5. Secrets (same values locally in `.dev.vars` for local Worker only; production tokens are Wrangler/Actions secrets):

| Secret | Where | Used for |
|---|---|---|
| `ADMIN_TOKEN` | `npx wrangler secret put ADMIN_TOKEN` | Bearer token for `/api/admin/*` and the `/admin` UI |
| `INGEST_TOKEN` | `npx wrangler secret put INGEST_TOKEN` | Worker side of collector ingest |
| `SOURCE_WATCH_INGEST_TOKEN` | GitHub Actions secret **and** local env | Collector; **same value as** `INGEST_TOKEN` |
| `GITHUB_DISPATCH_TOKEN` | `npx wrangler secret put GITHUB_DISPATCH_TOKEN` | Worker cron / Refresh now → `workflow_dispatch` (`actions:write`) |
| `GITHUB_TOKEN` | Actions provides this | Collector GitHub API |

6. `wrangler.jsonc` `vars.GITHUB_DISPATCH_REPO` is already `macgyver13/silent-payments-watch`. Keep `GITHUB_DISPATCH_WORKFLOW=refresh-feed.yml` and `GITHUB_DISPATCH_REF=main`.
7. `python3 scripts/sync_hugo_content.py` (already a service Hugo shell after promote).
8. Deploy: dashboard Build command `hugo --source site --minify`, Deploy command `npx wrangler deploy`, `HUGO_VERSION=0.164.0`. Or locally:

```bash
hugo --source site --minify
npx wrangler deploy
```

9. `serving.service_url` and `base_url` are `https://silent-payments-watch.macgyver-dev.workers.dev/`. After changing them, re-sync Hugo and redeploy so assets match.
10. First ingest (from a machine with the token, or Actions `workflow_dispatch`). D1 is empty, so the collector reads `data/public/` and preserves discovery dates:

```bash
SOURCE_WATCH_INGEST_TOKEN=… GITHUB_TOKEN=$(gh auth token) python3 scripts/build_seed_feed.py
python3 scripts/verify_public_artifacts.py
```

`--seed-only` in service mode requires `--allow-partial-ingest` and will not pick up live candidates.

`GET /admin` is public chrome; APIs need `ADMIN_TOKEN`. Failed admin auths: 10 per IP per minute, then 429. Hide/exclude update the public feed immediately; include terms and seed additions apply on the next collect.

## Refresh

The Worker cron (`27 2 * * *`, 02:27 UTC daily) POSTs GitHub `workflow_dispatch` for `.github/workflows/refresh-feed.yml`. That workflow runs the collector and ingests; it commits nothing. `vars.REFRESH_CRON` must match that expression so leftover Cloudflare cadences are skipped. If `GITHUB_DISPATCH_TOKEN` is empty, cron writes `refresh_skipped` and `/admin` shows `refresh not configured`. Staggered 10 minutes after frost-watch so the two collectors do not share the same GitHub/Delving burst.

Manual:

```bash
SOURCE_WATCH_INGEST_TOKEN=… GITHUB_TOKEN=$(gh auth token) python3 scripts/build_seed_feed.py
```

## Local Worker

Put `ADMIN_TOKEN=devadmin` and `INGEST_TOKEN=devingest` in `.dev.vars` (gitignored). Temporarily set `serving.service_url: "http://localhost:8787/"`.

```bash
python3 scripts/sync_hugo_content.py
npx wrangler d1 migrations apply silent-payments-watch --local
npx wrangler dev --test-scheduled
# other shell:
SOURCE_WATCH_INGEST_TOKEN=devingest \
  python3 scripts/build_seed_feed.py --seed-only --allow-partial-ingest
```

Open http://localhost:8787/ and http://localhost:8787/admin (token `devadmin`). Live collect: `GITHUB_TOKEN=$(gh auth token) SOURCE_WATCH_INGEST_TOKEN=devingest python3 scripts/build_seed_feed.py`.

## Config

- `config/watch.yaml` — instance identity, relevance, `discovered_after`, `serving` (`mode: service`; `service_url` required before ingest).
- `config/source-seeds.yaml` — seeded sources and live collectors. Pipeline input only; not read at request time.

## Tests

```bash
python3 -m unittest discover -s tests -v
npm test
npm run smoke:d1
```
