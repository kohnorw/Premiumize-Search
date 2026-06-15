# premiumize-web

A self-hosted bridge between Premiumize.me and your media stack (Radarr, Sonarr, Plex). It exposes a Torznab indexer and a qBittorrent-compatible download client, both backed by Premiumize's cache. Cached torrents appear instantly in a FUSE virtual filesystem — no downloading required.

---

## How it works

```
Radarr / Sonarr
  │
  ├─► Torznab indexer  →  Prowlarr (searches indexers)
  │       └─ results filtered to Premiumize-cached hashes only
  │
  └─► qBittorrent mock  →  Premiumize API
          └─ cached torrents → FUSE mount (instant)
             uncached torrents → Premiumize download queue
```

1. **Search** — Radarr/Sonarr query the built-in Torznab endpoint. The app forwards the search to Prowlarr, checks every result's info hash against the Premiumize cache, and returns only the cached hits (configurable).
2. **Grab** — Radarr/Sonarr send the chosen release to the qBittorrent mock. If it's cached on Premiumize it's marked ready immediately. If not, it's queued for Premiumize to download.
3. **Stream** — A FUSE filesystem mounts the cached content at `/docker/premiumize-web/mnt/`. Plex (or any other player) reads files directly from there via Premiumize CDN links, with block-level caching to avoid re-fetching the same chunks.

---

## Requirements

- Docker with `privileged` mode (required for FUSE)
- A Premiumize.me account with an API key
- Prowlarr (for indexer searches — optional but strongly recommended)
- Radarr and/or Sonarr (optional)
- Plex or similar (optional)

---

## First-time setup

### 1. Prepare the host mount

Run once on the host as root before starting the container:

```bash
cd /docker/premiumize-web
sudo ./setup.sh
```

This makes `/docker/premiumize-web` a shared bind mount so the FUSE filesystem inside the container propagates to the host. It installs a systemd service to persist this across reboots.

### 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env` and set at minimum:

```env
PREMIUMIZE_API_KEY=your_premiumize_api_key_here
TORZNAB_APIKEY=changeme        # change this — used by Prowlarr to authenticate
PROWLARR_URL=http://192.168.1.x:9696
PROWLARR_API_KEY=your_prowlarr_api_key
```

### 3. Start the container

```bash
docker compose up -d --build
```

The web UI is available at `http://<your-server-ip>:5000`.

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `PREMIUMIZE_API_KEY` | — | **Required.** Your Premiumize API key. |
| `TORZNAB_APIKEY` | `changeme` | API key Prowlarr uses to authenticate against the Torznab endpoint. Change this. |
| `PROWLARR_URL` | — | Full URL to your Prowlarr instance, e.g. `http://192.168.1.10:9696`. Can also be set in the UI. |
| `PROWLARR_API_KEY` | — | Prowlarr API key. Can also be set in the UI. |
| `TORZNAB_RATE_LIMIT` | `60` | Max Torznab requests per rate window. |
| `TORZNAB_RATE_WINDOW` | `60` | Rate limit window in seconds. |
| `ENABLE_FUSE` | `true` | Set to `false` to disable the FUSE mount (useful for testing). |

Environment variables take precedence over values saved through the UI.

---

## Prowlarr configuration

Add the app as a custom Torznab indexer in Prowlarr:

| Setting | Value |
|---|---|
| URL | `http://<your-server-ip>:5000/torznab` |
| API Key | Value of `TORZNAB_APIKEY` (default: `changeme`) |
| Categories | Movies (2000), TV (5000), Other (8000) |

Prowlarr will then include this indexer when Radarr/Sonarr trigger searches. Results are sourced from whichever indexers Prowlarr has configured (e.g. showRSS, TorrentDownload, LimeTorrents), filtered to Premiumize-cached hashes only.

---

## Radarr / Sonarr configuration

### Download client (qBittorrent mock)

Go to Settings → Download Clients → Add → qBittorrent:

| Setting | Value |
|---|---|
| Host | `<your-server-ip>` |
| Port | `5000` |
| URL Base | `/qbt` |
| Category | `radarr` (Radarr) or `tv-sonarr` (Sonarr) |
| Username | leave blank |
| Password | leave blank |

### Remote path mapping

Radarr/Sonarr need to know where completed downloads appear on disk:

| App | Remote Path | Local Path |
|---|---|---|
| Radarr | `/docker/premiumize-web/mnt/movies` | `/docker/premiumize-web/mnt/movies` |
| Sonarr | `/docker/premiumize-web/mnt/series` | `/docker/premiumize-web/mnt/series` |

### Indexer

Add the app's Torznab feed via Prowlarr (recommended) or directly:

| Setting | Value |
|---|---|
| URL | `http://<your-server-ip>:5000/torznab` |
| API Key | Value of `TORZNAB_APIKEY` |

---

## Plex configuration

Point your Plex libraries at the FUSE mount:

| Library type | Path |
|---|---|
| Movies | `/docker/premiumize-web/mnt/movies` |
| TV Shows | `/docker/premiumize-web/mnt/series` |

Files appear here as soon as a torrent is marked cached. Playback streams directly from Premiumize CDN via the built-in proxy, with block-level caching to speed up seeking and avoid redundant fetches.

---

## FUSE mount

The virtual filesystem mounts at `/docker/premiumize-web/mnt/` with two subdirectories:

```
/docker/premiumize-web/mnt/
  movies/    ← content without season/episode patterns in the name
  series/    ← content with S01E01 / Season 1 / 1x01 patterns
  cache/     ← persistent block cache (managed automatically)
```

Torrent names are used to auto-detect movies vs series. The category set by Sonarr/Radarr is used as a fallback when the name alone is ambiguous.

### FUSE settings (UI)

Advanced mount settings are available in the web UI under **Settings → Mount**:

| Setting | Default | Description |
|---|---|---|
| Cache directory | `/docker/premiumize-web/cache` | Where block cache files are stored on disk. |
| Disk cache size | `30 GB` | Maximum disk space used by the block cache. Oldest blocks are evicted first. |
| Buffer memory | `512 MB` | In-memory block buffer size. |
| Chunk size | `8 MB` | Size of each cached block. |
| Read-ahead | `500 MB` | How much to prefetch ahead of the current read position. |
| Cache expiry | `24h` | How long blocks are kept on disk before expiry. |
| Cache cleanup | `20m` | How often the cleanup job runs. |
| Prefetch | enabled | Prefetch blocks ahead of playback position. |
| Video only | enabled | Only cache video file extensions; skip subtitles, NFOs, etc. |

---

## Web UI

The web UI (`http://<your-server-ip>:5000`) provides:

- **Search** — search Prowlarr/TPB directly, see which results are cached on Premiumize, and add them to your library with one click.
- **Library** — browse all content currently available in the FUSE mount.
- **Settings** — configure Premiumize API key, Prowlarr connection, Torznab options, and FUSE mount parameters.
- **Backup / Restore** — export and import your cached transfer list as JSON.

---

## Torznab API

The Torznab endpoint is at `/torznab` (also accessible at `/torznab/api`):

| Parameter | Description |
|---|---|
| `t=caps` | Returns capability XML (no auth required). |
| `t=search&q=<query>` | General search. |
| `t=tvsearch&q=<title>&season=<n>&ep=<n>` | TV search with season/episode. |
| `t=movie&q=<title>` | Movie search. |
| `apikey=<key>` | Authentication (matches `TORZNAB_APIKEY`). |

Supported categories:

| ID | Name |
|---|---|
| 2000 | Movies |
| 2010–2080 | Movie subcategories (SD, HD, UHD, BluRay, WEB-DL, etc.) |
| 5000 | TV |
| 5010–5090 | TV subcategories (WEB-DL, SD, HD, UHD, Anime, etc.) |
| 8000 | Other |

Magnet links returned by the Torznab feed include tracker announce URLs so clients with DHT disabled can connect.

---

## Cached-only mode

By default the Torznab feed returns all results from Prowlarr that have an extractable info hash, with cached items sorted to the top. You can restrict to cached-only results (hiding anything not on Premiumize) via the UI under **Settings → Torznab → Cached only**, or via the API:

```bash
curl -X POST http://localhost:5000/api/torznab/cached-only \
  -H "Content-Type: application/json" \
  -d '{"cached_only": true}'
```

When enabled and a search returns no cached results, the full result set is returned as a fallback to avoid empty feeds confusing Radarr/Sonarr.

---

## Backup and restore

Export your library:

```bash
curl http://localhost:5000/api/backup -o premiumize-backup.json
```

Restore on a new instance:

```bash
curl -X POST http://localhost:5000/api/restore \
  -F "file=@premiumize-backup.json"
```

Restore only imports hashes that are currently cached on Premiumize. Hashes no longer on Premiumize are skipped.

---

## Troubleshooting

**No search results**
- Check Prowlarr is reachable from inside the container: `docker exec premiumize-web python3 -c "import urllib.request; print(urllib.request.urlopen('http://<prowlarr-ip>:9696/api/v1/indexer', timeout=5).status)"`
- Confirm the Prowlarr URL saved in the app points to the correct host — from inside Docker, use the host's LAN IP or `host.docker.internal`, not `localhost`.
- Check the Premiumize API key is set — without it the cache check is skipped and all results appear uncached.

**Red cloud / download fails in Radarr/Sonarr**
- Confirm the download client in Radarr/Sonarr is pointing to the qBittorrent mock (`host: <server-ip>`, `port: 5000`, `url base: /qbt`).
- Check `docker logs premiumize-web` for errors at the moment you click download.

**FUSE mount empty**
- Run `docker logs premiumize-web | grep FUSE` to check mount status.
- Ensure `setup.sh` was run on the host before the container started.
- Check the container is running with `privileged: true` and `/dev/fuse` device access.

**Content not showing in Plex**
- Trigger a Plex library scan after adding content.
- Confirm Plex has read access to `/docker/premiumize-web/mnt/`.

---

## Uninstall

```bash
cd /docker/premiumize-web
docker compose down --rmi all
umount -l /docker/premiumize-web/mnt 2>/dev/null || true
systemctl stop premiumize-web-rshared.service
systemctl disable premiumize-web-rshared.service
rm -f /etc/systemd/system/premiumize-web-rshared.service
systemctl daemon-reload
rm -rf /docker/premiumize-web
```
