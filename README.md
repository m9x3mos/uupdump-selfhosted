# UUP Dump Self-Hosted

A self-hosted [UUP dump](https://uupdump.net) instance packaged as a single
Docker image: Nginx + PHP-FPM 8.2 serving the UUP dump JSON API and a local
web frontend. Download Windows Update packages (ESDs/CABs) for any build,
language, and edition, then convert them with the bundled converter tooling.

- **Image:** `m9x3mos/uupdump-selfhosted`
- **Tags:** `latest`
- **License:** MIT

## What's included

- **Backend/API** – UUP dump JSON API (`/api/`) backed by the upstream core,
  including `listid`, `fetchupd`, `get`, `listlangs`, `listeditions`.
- **Frontend** – UUP dump–style web UI: known builds browser, fetch latest
  build, language/edition selection, file details, download package generation,
  FAQ/about, dark mode.
- **Converter & standalone tools** – `converter/` and `standalone/` CLIs for
  offline package conversion.
- **Sidecars** – optional periodic build discovery (`cron`) and background language/edition pack prewarming (`packworker`).
- **TrueNAS SCALE** support out of the box (PUID/PGID/ACL handling).

## Quick start (Docker Compose)

```sh
mkdir uupdump && cd uupdump
# fetch the compose file (or copy it from the repo)
curl -O https://raw.githubusercontent.com/.../docker-compose.yml
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
  -e PUID=33 -e PGID=33 \
  --restart unless-stopped \
  m9x3mos/uupdump-selfhosted:latest
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
      # Generated language/edition packs (written by standalone pack generator)
      - ./packs:/var/www/html/api/packs
      # API/file cache (rate limits, request caches)
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
      # All mounted writes use one numeric identity. Defaults match normal Linux
      # Docker (www-data); override on TrueNAS to match the Apps ACL (568:568).
      PUID: 33
      PGID: 33
      CHOWN_MOUNTS: 1
    restart: unless-stopped

  # Optional: periodically fetch new builds into the shared fileinfo database
  uupdump-cron:
    image: m9x3mos/uupdump-selfhosted:latest
    container_name: uupdump-cron
    depends_on:
      - uupdump
    volumes:
      - ./fileinfo:/var/www/html/api/fileinfo
      - ./cache:/var/www/html/api/cache
      - ./packs:/var/www/html/api/packs
      - ./config:/var/www/html/api/config
      - ./uuptmp:/var/www/html/api/uuptmp
    environment:
      UUP_PUBLIC_URL: http://localhost:8081
      PUID: 33
      PGID: 33
      CHOWN_MOUNTS: 1
      # Optional overrides (auto-detected by default):
      # SCAN_REVISION_FAMILY: 26300
      # SCAN_REVISION_COUNT: 10
      # SCAN_CYCLE_SECONDS: 21600
    # This container runs the scan loop (no local web server). A heartbeat file
    # is updated throughout cycles and during long waits to signal liveness.
    healthcheck:
      test: ["CMD-SHELL", "t=$$(cat /var/www/html/api/cache/cron.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 900 ]"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 120s
    entrypoint: ["/usr/local/bin/uup-entrypoint.sh", "cron"]
    restart: unless-stopped

  # Prewarm language/edition packs in the background so page loads never have
  # to download metadata while the user is waiting on a selection screen.
  uupdump-packworker:
    image: m9x3mos/uupdump-selfhosted:latest
    container_name: uupdump-packworker
    depends_on:
      - uupdump
    volumes:
      - ./fileinfo:/var/www/html/api/fileinfo
      - ./cache:/var/www/html/api/cache
      - ./packs:/var/www/html/api/packs
      - ./uuptmp:/var/www/html/api/uuptmp
    environment:
      TZ: UTC
      PUID: 33
      PGID: 33
      CHOWN_MOUNTS: 1
      PACKWORKER_INTERVAL: 20
    # This container runs the prewarm loop (no local web server). A heartbeat
    # file is updated each pass/build attempt to signal liveness.
    healthcheck:
      test: ["CMD-SHELL", "t=$$(cat /var/www/html/api/cache/packworker.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 600 ]"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    entrypoint: ["/usr/local/bin/uup-entrypoint.sh", "packworker"]
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
      PUID: 568
      PGID: 568
      CHOWN_MOUNTS: 0
    restart: unless-stopped

  uupdump-cron:
    image: m9x3mos/uupdump-selfhosted:latest
    container_name: uupdump-cron
    depends_on:
      - uupdump
    volumes:
      - ./fileinfo:/var/www/html/api/fileinfo
      - ./cache:/var/www/html/api/cache
      - ./packs:/var/www/html/api/packs
      - ./config:/var/www/html/api/config
      - ./uuptmp:/var/www/html/api/uuptmp
    environment:
      UUP_PUBLIC_URL: http://localhost:8081
      PUID: 568
      PGID: 568
      CHOWN_MOUNTS: 0
      # SCAN_REVISION_FAMILY: 26300
      # SCAN_REVISION_COUNT: 10
      # SCAN_CYCLE_SECONDS: 21600
    # This container runs the scan loop (no local web server). A heartbeat file
    # is updated throughout cycles and during long waits to signal liveness.
    healthcheck:
      test: ["CMD-SHELL", "t=$$(cat /var/www/html/api/cache/cron.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 900 ]"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 120s
    entrypoint: ["/usr/local/bin/uup-entrypoint.sh", "cron"]
    restart: unless-stopped

  uupdump-packworker:
    image: m9x3mos/uupdump-selfhosted:latest
    container_name: uupdump-packworker
    depends_on:
      - uupdump
    volumes:
      - ./fileinfo:/var/www/html/api/fileinfo
      - ./cache:/var/www/html/api/cache
      - ./packs:/var/www/html/api/packs
      - ./uuptmp:/var/www/html/api/uuptmp
    environment:
      TZ: UTC
      PUID: 568
      PGID: 568
      CHOWN_MOUNTS: 0
      PACKWORKER_INTERVAL: 20
    # This container runs the prewarm loop (no local web server). A heartbeat
    # file is updated each pass/build attempt to signal liveness.
    healthcheck:
      test: ["CMD-SHELL", "t=$$(cat /var/www/html/api/cache/packworker.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 600 ]"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    entrypoint: ["/usr/local/bin/uup-entrypoint.sh", "packworker"]
    restart: unless-stopped
```

## Persistent volumes

| Host path | Container path | Purpose |
| --- | --- | --- |
| `./fileinfo` | `/var/www/html/api/fileinfo` | Build database |
| `./packs` | `/var/www/html/api/packs` | Generated language/edition packs |
| `./cache` | `/var/www/html/api/cache` | API/file cache, rate limits |
| `./config` | `/var/www/html/api/config` | Revision seed + scan settings (survives data clears) |
| `./downloads` | `/var/www/html/downloads` | Downloaded artifacts |
| `./uuptmp` | `/var/www/html/api/uuptmp` | Nginx/PHP scratch space |

## Environment variables

| Variable | Default | Description |
| --- | --- | --- |
| `UUP_PUBLIC_URL` | `http://localhost:8081` | Public base URL used in generated download packages/scripts. Override for reverse proxy / remote access (port is preserved). |
| `PUID` | `33` | UID for PHP-FPM, nginx, and sidecar writes |
| `PGID` | `33` | GID paired with `PUID` |
| `CHOWN_MOUNTS` | `1` | Recursively chown mounted writable dirs to `PUID:PGID` on startup (set `0` on hosts that manage ACLs, e.g. TrueNAS) |
| `PACKWORKER_INTERVAL` | `20` | Seconds between pack prewarmer passes (queue drain + recent-build prewarm) |
| `TZ` | `UTC` | Container timezone |

## Cron sidecar

Periodically discovers the newest build on Retail/RP/Beta/Dev rings and refreshes recent revisions of the current build family. The container runs the scan loop (no web server), so its healthcheck reads a heartbeat file written throughout each cycle and during long waits.

```yaml
uupdump-cron:
  image: m9x3mos/uupdump-selfhosted:latest
  depends_on: [uupdump]
  volumes:
    - ./fileinfo:/var/www/html/api/fileinfo
    - ./cache:/var/www/html/api/cache
    - ./packs:/var/www/html/api/packs
    - ./config:/var/www/html/api/config
    - ./uuptmp:/var/www/html/api/uuptmp
  environment:
    UUP_PUBLIC_URL: http://localhost:8081
    PUID: 33
    PGID: 33
    CHOWN_MOUNTS: 1
  healthcheck:
    test: ["CMD-SHELL", "t=$$(cat /var/www/html/api/cache/cron.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 900 ]"]
    interval: 60s
    timeout: 10s
    retries: 3
    start_period: 120s
  entrypoint: ["/usr/local/bin/uup-entrypoint.sh", "cron"]
  restart: unless-stopped
```

Overrides: `SCAN_REVISION_FAMILY`, `SCAN_REVISION_COUNT`, `SCAN_RING_GAP_SECONDS`, `SCAN_CYCLE_SECONDS`, `SCAN_ONCE=1` (single cycle, useful for testing).

## Pack prewarmer sidecar

Prewarms language/edition packs in the background (drains the queue written on fetch and tops up recent builds missing valid packs) so selection screens do not generate metadata on demand. The container runs the prewarm loop (no web server), so its healthcheck reads a heartbeat file written each pass/build attempt.

```yaml
uupdump-packworker:
  image: m9x3mos/uupdump-selfhosted:latest
  depends_on: [uupdump]
  volumes:
    - ./fileinfo:/var/www/html/api/fileinfo
    - ./cache:/var/www/html/api/cache
    - ./packs:/var/www/html/api/packs
    - ./uuptmp:/var/www/html/api/uuptmp
  environment:
    TZ: UTC
    PUID: 33
    PGID: 33
    CHOWN_MOUNTS: 1
    PACKWORKER_INTERVAL: 20
  healthcheck:
    test: ["CMD-SHELL", "t=$$(cat /var/www/html/api/cache/packworker.heartbeat 2>/dev/null || echo 0); [ $$(( $$(date +%s) - t )) -le 600 ]"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 60s
  entrypoint: ["/usr/local/bin/uup-entrypoint.sh", "packworker"]
  restart: unless-stopped
```

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
- `config/scan.json` – `{"family": "auto", "count": 10}`.

Manage both in the app under **New build → Revision seed**, or via
`GET/POST /api/seed.php`. Cron settings precedence: env vars → `scan.json` →
built-in defaults.

## Notes

- The image is a feature-parity work-in-progress vs. public uupdump.net;
  end-to-end conversion/download fidelity is still being validated.
- For public hosting, add auth, rate limiting, TLS, and log management.
- API endpoints and JSON schemas are documented in the repo's `readme.md`.
```<|end|><|start|>tool to=assistant<|channel|>analysis<|message|>I wrote the file. Let me verify it exists and reads back.<|end|><|start|><|start|>assistant to=read<|channel|>analysis <|constrain|>xml<|message|><rfl:parameter name="path">
C:\Users\BillD\Documents\UUP\dockerhub.md
