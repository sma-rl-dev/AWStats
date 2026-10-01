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
| `deploy` | Build `Dockerfile.tester-env` from editable source, start container, build month/day/hour databases |
| `seed` | Clear `/var/lib/awstats/awstats*.txt` and re-run native `awstats.pl -update` for `month` (default), `-databasebreak=day`, and `-databasebreak=hour` |
| `verify` | Assert the report page contains `Statistics for analytics.example.test (2025-01)` and the month (`awstats012025`), Day 15 (`awstats01202515`), and Jan-18-hour-10 (`awstats0120251810`) history files exist and are non-empty |
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

- Fixed eighteen-line combined-log fixture at `deployment/access.log`
  (January 15–20, 2025; deterministic IPs, dates, UAs, referrers, statuses).
  Lines 1–8 (Jan 15–17): homepage/product/pricing/docs traffic, two external
  referrers, three browser families, one normal 404; top tied pages
  `/products` x2 and `/docs/getting-started` x2.
  Lines 9–16 (Jan 18–19, one line each): `.webp` asset hit (NotPageList),
  GPTBot robot hit, authenticated `user1` hit, Edge/12.10136 hit, Android 13
  hit, `picks.yahoo.com` referrer hit, HTTP 101 hit, HTTP 206 + Googlebot hit
  on `/downloads/annual-report.pdf` (download extension so the 206 takes the
  robot-detection path).
  Lines 17–18 (Jan 20): `.woff2` asset hit (NotPageList) and HTTP 451 hit
  (renders with the legacy IIS label).
- Site config `deployment/awstats.tester-env.conf` sets
  `ShowAuthenticatedUsers=PHBL` so the seeded login row is visible.
- `seed` clears `/var/lib/awstats/awstats*.txt`, then runs three native
  updates for config `tester-env` (SiteDomain
  `analytics.example.test`, `DirData=/var/lib/awstats`):
  `-update` (month database), `-databasebreak=day -update` (one history
  file per day, Jan 15–20), `-databasebreak=hour -update` (one history
  file per active hour: Jan-15-09, Jan-16-11, Jan-17-14, Jan-18-10,
  Jan-19-09, Jan-20-10). Re-running `seed` reproduces identical totals
  (each break reprocesses the full fixture exactly once; no double
  counting). No fake HTML or hand-built databases; every history file is
  produced by `awstats.pl` itself.
- Report totals: 8 unique visitors, 8 visits, 10 pages, 13 hits, 24.00 KB.
  February 2025 remains empty.
- Day/hour scopes are served from their native databases and carry
  reduced nonzero totals: Day 15 (`databasebreak=day&day=15`) shows
  2 visitors, 2 visits, 3 pages, 3 hits, 5.00 KB with 2 different
  pages-url (`/products` x2, `/` x1), First/Last visit
  15 Jan 2025 09:00–09:08; Jan-18 hour 10
  (`databasebreak=hour&day=18&hour=10`) shows 3 visitors, 3 visits,
  3 pages, 4 hits, 6.00 KB with 3 different pages-url
  (`/account/profile`, `/blog/edge-release`, `/blog/android-app` x1 each;
  the 10:00 `.webp` asset contributes a non-page hit and the 10:05 GPTBot
  robot hit is not-viewed traffic), First/Last visit 18 Jan 2025 10:10–10:20.
  The Reported period controls retain the selected granularity/day/hour and the hour selector
  offers 0–23. Known native rendering fossil (no source change made):
  the Summary period cell still reads `Month Jan 2025` in day/hour views;
  scope is established by the controls, URL parameters, First/Last visit
  range, and reduced nonzero totals.

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
