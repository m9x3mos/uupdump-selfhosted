# UUP Dump Self-Hosted

A self-hosted [UUP dump](https://uupdump.net) instance packaged as a single
Docker image: Nginx + PHP-FPM 8.2 serving the UUP dump JSON API and a local
web frontend. Download Windows Update packages (ESDs/CABs) for any build,
language, and edition, then convert them with the bundled converter tooling.

- **GitHub:** https://github.com/m9x3mos/uupdump-selfhosted
- **Docker image:** https://hub.docker.com/r/m9x3mos/uupdump-selfhosted/
- **Issues:** please report bugs and feature requests on [GitHub Issues](https://github.com/m9x3mos/uupdump-selfhosted/issues)
- **Image:** `m9x3mos/uupdump-selfhosted`
- **Tags:** `1.1.0`, `latest`
- **License:** MIT

## What's included

- **Backend/API** – UUP dump JSON API (`/api/`) backed by the upstream core,
  including `listid`, `fetchupd`, `get`, `listlangs`, `listeditions`, plus the local
  `activefamilies`, `seed` and `releaseinfo` endpoints used by the UI.
- **Frontend** – UUP dump–style web UI: known builds browser, fetch latest
  build, language/edition selection, file details, download package generation,
  FAQ/about, dark mode.
- **Active family discovery** – Microsoft Flight Hub current builds and release-health
  data seed automatic scans; tracked families persist in `./config/scan.json`.
  Family-only lookups probe current Retail/Insider revisions without writing exact
  revisions into `config/revisions.seed`.
- **Converter & standalone tools** – `converter/` and `standalone/` CLIs for
  offline package conversion.
- **Background services (v1.1.0+)** – build discovery and language/edition pack prewarming now run inside the same container as the web server, supervised with tagged logs and per-child watchdogs. No sidecar containers needed.
- **TrueNAS SCALE** support out of the box (PUID/PGID/ACL handling).

## Quick start (Docker Compose)

```sh
mkdir uupdump && cd uupdump
# fetch the compose file (or copy it from the repo)
curl -O https://raw.githubusercontent.com/m9x3mos/uupdump-selfhosted/master/docker-compose.yml
docker compose up -d
```

Then open **http://localhost:8081**.

App routes: `/#/`, `/#/known`, `/#/fetchupd`, `/#/selectlang`,
`/#/download`, `/#/faq`, `/#/about`. API health: `/api/index.php`.

### docker run (minimal)

```sh
docker run -d --name uupdump-selfhosted \
  -p 8081:80 \
  -v ./fileinfo:/var/www/html/api/fileinfo \
  -v ./cache:/var/www/html/api/cache \
  -v ./packs:/var/www/html/api/packs \
  -v ./config:/var/www/html/api/config \
  -v ./downloads:/var/www/html/downloads \
  -v ./uuptmp:/var/www/html/api/uuptmp \
  -e UUP_PUBLIC_URL=http://localhost:8081 \
  -e UUP_API_BASE=http://127.0.0.1/api \
  -e PUID=33 -e PGID=33 \
  --entrypoint /usr/local/bin/uup-entrypoint.sh \
  --restart unless-stopped \
  m9x3mos/uupdump-selfhosted:1.1.0 all
```

## Full Docker Compose example (localhost)

```yaml
services:
  uupdump:
    image: m9x3mos/uupdump-selfhosted:latest
    container_name: uupdump-selfhosted
    ports:
      - "8081:80"
    volumes:
      # Persistent fileinfo database (mounted where the API reads it)
      - ./fileinfo:/var/www/html/api/fileinfo
      # Generated language/edition packs (written by the pack worker)
      - ./packs:/var/www/html/api/packs
      # API/file cache, request caches and loop heartbeats
      - ./cache:/var/www/html/api/cache
      # UI-managed revision seed / scan settings (survives data clears)
      - ./config:/var/www/html/api/config
      # Optional: store downloaded artifacts here
      - ./downloads:/var/www/html/downloads
      # nginx/PHP scratch space that must be writable by the runtime UID
      - ./uuptmp:/var/www/html/api/uuptmp
    environment:
      TZ: UTC
      # Public base URL used in generated download packages/scripts (override for reverse proxy or remote access)
      UUP_PUBLIC_URL: http://localhost:8081
      # The in-container scanner calls this container's own web server
      UUP_API_BASE: http://127.0.0.1/api
      # All mounted writes use one numeric identity. Defaults match normal Linux
      # Docker (www-data); override on TrueNAS to match the Apps ACL (568:568).
      PUID: 33
      PGID: 33
      CHOWN_MOUNTS: 1
      PACKWORKER_INTERVAL: 20
      # Optional scan overrides (auto-detected by default):
      # SCAN_REVISION_FAMILY: 26300
      # SCAN_REVISION_COUNT: 10
      # SCAN_CYCLE_SECONDS: 21600
    # One container runs web, FPM, build scanner and pack prewarmer. The
    # entrypoint supervises all four with tagged logs and restarts any child
    # that exits unexpectedly.
    entrypoint: ["/usr/local/bin/uup-entrypoint.sh", "all"]
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1/api/index.php >/dev/null || exit 1; t=$$(cat /var/www/html/api/cache/cron.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 900 ] || exit 1; t=$$(cat /var/www/html/api/cache/packworker.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 600 ]"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 180s
    restart: unless-stopped
```

## TrueNAS SCALE compose example

```yaml
services:
  uupdump:
    image: m9x3mos/uupdump-selfhosted:latest
    container_name: uupdump-selfhosted
    ports:
      - "8081:80"
    volumes:
      - ./fileinfo:/var/www/html/api/fileinfo
      - ./packs:/var/www/html/api/packs
      - ./cache:/var/www/html/api/cache
      - ./config:/var/www/html/api/config
      - ./downloads:/var/www/html/downloads
      - ./uuptmp:/var/www/html/api/uuptmp
    environment:
      TZ: UTC
      UUP_PUBLIC_URL: http://localhost:8081
      UUP_API_BASE: http://127.0.0.1/api
      PUID: 568
      PGID: 568
      CHOWN_MOUNTS: 0
    entrypoint: ["/usr/local/bin/uup-entrypoint.sh", "all"]
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1/api/index.php >/dev/null || exit 1; t=$$(cat /var/www/html/api/cache/cron.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 900 ] || exit 1; t=$$(cat /var/www/html/api/cache/packworker.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 600 ]"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 180s
    restart: unless-stopped
```

## Persistent volumes

| Host path | Container path | Purpose |
| --- | --- | --- |
| `./fileinfo` | `/var/www/html/api/fileinfo` | Build database |
| `./packs` | `/var/www/html/api/packs` | Generated language/edition packs |
| `./cache` | `/var/www/html/api/cache` | API/file cache, rate limits |
| `./config` | `/var/www/html/api/config` | Revision seed, scan settings + persisted tracked families (survives data clears) |
| `./downloads` | `/var/www/html/downloads` | Downloaded artifacts |
| `./uuptmp` | `/var/www/html/api/uuptmp` | Nginx/PHP scratch space |

## Environment variables

| Variable | Default | Description |
| --- | --- | --- |
| `UUP_PUBLIC_URL` | `http://localhost:8081` | Public base URL used in generated download packages/scripts. Override for reverse proxy / remote access (port is preserved). |
| `UUP_API_BASE` | `http://127.0.0.1/api` | API base URL used by the in-container build scanner. Leave as-is unless you intentionally run the scanner against another instance. |
| `PUID` | `33` | UID for PHP-FPM, nginx, and background-loop writes |
| `PGID` | `33` | GID paired with `PUID` |
| `CHOWN_MOUNTS` | `1` | Recursively chown mounted writable dirs to `PUID:PGID` on startup (set `0` on hosts that manage ACLs, e.g. TrueNAS) |
| `PACKWORKER_INTERVAL` | `20` | Seconds between pack prewarmer passes (queue drain + recent-build prewarm) |
| `SCAN_REVISION_FAMILY` / `SCAN_REVISION_COUNT` / `SCAN_CYCLE_SECONDS` | auto / 10 / 21600 | Optional build-scanner overrides |
| `TZ` | `UTC` | Container timezone |

## Background services and supervision (v1.1.0+)

The three former services (`uupdump`, `uupdump-cron`, `uupdump-packworker`) are now a single container started with the `all` entrypoint mode. It supervises four children:

| Log tag | Process | Job |
| --- | --- | --- |
| `[fpm]` | PHP-FPM (foreground) | Executes the JSON API |
| `[web]` | nginx | Serves the SPA and API |
| `[cron]` | `cron/scan.sh` | Refreshes Microsoft Flight Hub data, scans pinned/active families plus persisted tracked families, and refreshes seeded exact revisions |
| `[packworker]` | `cron/packworker.sh` | Drains the pack queue and tops up recent builds |

Every line in `docker logs` is tagged with its child name. If any child exits unexpectedly, the watchdog logs the status and restarts it after a 2-second backoff. A readiness gate waits for the local API before starting the loops, and TERM/INT stops all children cleanly.

Liveness is reported through `cache/cron.heartbeat` and `cache/packworker.heartbeat`; the single compose healthcheck checks the web API and both heartbeat files.

Scanner overrides: `SCAN_REVISION_FAMILY`, `SCAN_REVISION_COUNT`, `SCAN_RING_GAP_SECONDS`, `SCAN_CYCLE_SECONDS`, and `SCAN_ONCE=1` (single cycle, useful for testing).

For compatibility, the image still supports the old `web`, `cron`, and `packworker` entrypoint modes if you deliberately want to run separate containers.

## Browser caching (v1.1.0+)

nginx sends `Cache-Control: no-cache` on every response so the browser revalidates via `ETag` and receives `304` when nothing changed. UI API requests use `cache: 'no-store'`, so build lists and job status are never served from the HTTP cache. **Refresh** in the header (**Refresh data** in the sidebar) additionally clears the SPA's in-memory caches and app storage, then reloads with a one-time cache-buster. Use it if information ever looks stale.

## Microsoft release workflow (v1.1.0+)

Under **New build → Microsoft release information**:

- **Scrape only** saves the release snapshot without changing the seed/database.
- **Scrape, seed and fetch** merges revisions into the seed and fetches each build for `amd64` and `arm64`.
- **Pull all release families** (off by default), **Pull all found versions** (off by default, first 10 per family for the scrape/saved snapshot, displayed list, seed and fetch) and **Skip builds already known** (on by default) control the scope.
- Saved snapshots feed active-family discovery; release rows are sorted by normalized availability dates, so date-only and timestamped values share one timeline.
- **Stop** cancels after the current task and preserves the stop state across worker writes; the found-build list remains visible while jobs run.

## Active family discovery (v1.1.0+)

- The UI reads Microsoft Flight Hub Markdown from `https://learn.microsoft.com/en-us/windows-insider/flight-hub/?accept=text/markdown`. Results are cached for 10 minutes; `?refresh=1` forces a fresh lookup, and the last good cache is used if Microsoft is unavailable. Only bold/current rows are active; historical rows such as `26300.7674` are ignored.
- **New build → Database scan → Scan active families** discovers families from Flight Hub, the saved release snapshot, the revision seed and local known builds, saves them for automatic checks, then starts one scan.
- **Save families for automatic checks** stores the current base-build list without scanning.
- Saved families live in `config/scan.json` and are scanned by cron on every cycle. Cron still emits up to `SCAN_REVISION_COUNT` exact revisions per seeded family. Family-only requests probe one newest current revision per source category (Retail, Insider, snapshot/seed/local fallbacks), de-duplicate and cap the list at 4.
- Discovery never writes exact revisions into `config/revisions.seed`, and the app makes no requests to `uupdump.net`.

## TrueNAS SCALE

```sh
docker compose -f docker-compose.yml -f docker-compose.truenas.yml up -d --build
```

Or set `PUID=568`, `PGID=568`, `CHOWN_MOUNTS=0` in the TrueNAS Apps form and
grant the `apps` user read/write/traverse ACLs on the mapped datasets. The
entrypoint detects non-root starts and runs everything as `568:568`.

## Revision seed (UI-managed)

Windows Update can't enumerate superseded revisions, so exact `major.minor`
numbers are stored in `./config`:

- `config/revisions.seed` – one revision per line, grouped by major build.
- `config/scan.json` – `{"family": "auto", "count": 10, "families": ["26300", "26200"]}`.

Manage the seed/family/count under **New build → Revision seed**, and save tracked
families under **New build → Database scan → Save families for automatic checks**.
`POST /api/seed.php` also accepts `action=setfamilies`. Cron settings precedence:
env vars → `scan.json` → built-in defaults.

## Notes

- The image is a feature-parity work-in-progress vs. public uupdump.net;
  end-to-end conversion/download fidelity is still being validated.
- Family scans use a 120-second per-request PHP limit; nginx's `fastcgi_read_timeout` is 600 seconds.
- For public hosting, add auth, rate limiting, TLS, and log management.
- API endpoints and JSON schemas are documented in the repo's `readme.md`.
