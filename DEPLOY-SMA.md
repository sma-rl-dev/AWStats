# AWStats — Deploy & Seed Notes (tester-env baseline)

## Source

- **Upstream:** https://github.com/eldy/awstats
- **Fork:** https://github.com/sma-rl-dev/awstats
- **Baseline branch:** `tester-env-baseline`
- **Upstream baseline:** tag `AWSTATS_8_0` (commit `4d50c5828ebbabab79f3ed7de52d5cd5299c1a1c`)
- **Docker image tag:** `tester-env-awstats:<hash>` (content-addressed per patch-set; manual invocations default to `:dev` via `IMAGE_TAG`)

## Quick Start

From the fork checkout root (`apps/awstats/` when used as a tester-env submodule):

```bash
./tester-env deploy    # Build image, start container, build report
./tester-env seed      # Rebuild the deterministic January 2025 report
./tester-env verify    # Check seeded report is served
./tester-env reset     # Clean slate (remove container + volume, keep image)
```

## CLI Commands

| Command | Description |
|---------|-------------|
| `deploy` | Build `Dockerfile.tester-env` from editable source, start container, build report |
| `seed` | Re-run `awstats.pl -config=tester-env -update` inside the container |
| `verify` | Assert the report page contains `Statistics for analytics.example.test (2025-01)` |
| `reset` | Remove container and data volume (`awstats-data[-<RUN_ID>]`); never removes the image |
| `stop` | Stop container (preserves data volume) |
| `logs` | Tail container logs |
| `status` | Show container status and URL |

Environment: `IMAGE_TAG` (content-addressed tag, defaults to `tester-env-awstats:dev`),
`RUN_ID` (scopes container and volume names only, never the image name),
`PORT` (default `18180`).

## Build

```bash
./tester-env deploy
```

- Base image `debian:bookworm-slim` with Apache CGI, Perl, `JSON::XS`, `Try::Tiny`.
- AWStats source copied to `/opt/awstats`; Apache conf from `deployment/apache-awstats.conf`;
  site config from `deployment/awstats.tester-env.conf`.
- Serves CGI at `/awstats/awstats.pl`; report URL:
  `http://localhost:18180/awstats/awstats.pl?config=tester-env&month=01&year=2025`
- No credentials, database, or external service required.

## Seed Data

```bash
./tester-env seed
```

- Fixed sixteen-line combined-log fixture at `deployment/access.log`
  (January 15–19, 2025; deterministic IPs, dates, UAs, referrers, statuses).
  Lines 1–8 (Jan 15–17): homepage/product/pricing/docs traffic, two external
  referrers, three browser families, one normal 404; top tied pages
  `/products` x2 and `/docs/getting-started` x2.
  Lines 9–16 (Jan 18–19, one line each): `.webp` asset hit (NotPageList),
  GPTBot robot hit, authenticated `user1` hit, Edge/12.10136 hit, Android 13
  hit, `picks.yahoo.com` referrer hit, HTTP 101 hit, HTTP 206 + Googlebot hit
  on `/downloads/annual-report.pdf` (download extension so the 206 takes the
  robot-detection path).
- Site config `deployment/awstats.tester-env.conf` sets
  `ShowAuthenticatedUsers=PHBL` so the seeded login row is visible.
- `seed` runs `awstats.pl -update` for config `tester-env` (SiteDomain
  `analytics.example.test`, `DirData=/var/lib/awstats`).
- Report totals: 8 unique visitors, 8 visits, 10 pages, 12 hits, 23.00 KB.
  February 2025 remains empty.

## Verify

```bash
./tester-env verify
```

- `curl` the report URL and grep for `Statistics for analytics.example.test (2025-01)`.

## Reset

```bash
./tester-env reset
```

- `docker rm -f` the container and `docker volume rm` the data volume; never `docker rmi`
  the image (reclaim with `scripts/rl-env gc --prune-images`).
- Never touches `alevern/chromium-browser` images or running `chromium-ls*` containers.

## Browser Smoke Evidence (2026-09-07, prospect-local, RUN_ID=awstats-accept)

- **URL:** `http://localhost:18180/awstats/awstats.pl?config=tester-env&month=01&year=2025`
- January 2025 report for `analytics.example.test` showed 4 unique visitors, 4 visits,
  6 pages, 7 hits, 15.00 KB with title `Statistics for analytics.example.test (2025-01) - main`.
- Using AWStats native Reported period controls, selected February 2025 and saw zero/NA
  metrics, then restored January and its original nonzero report.

## Port

- Default `18180`, configurable via `PORT`; `RUN_ID` isolates container
  (`tester-env-awstats[-<RUN_ID>]`) and volume (`awstats-data[-<RUN_ID>]`) names.
