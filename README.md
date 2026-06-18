<div align="center">

# 🎬 premiumize-web

**Stream your Premiumize library instantly. Search, grab, and auto-import into Radarr and Sonarr — without ever downloading a file to disk.**

premiumize-web is a self-hosted bridge between [Premiumize.me](https://premiumize.me) and your media stack. It mounts your Premiumize cloud library as a virtual filesystem, pretends to be a qBittorrent download client so Radarr and Sonarr import instantly, and keeps Plex/Jellyfin/Emby in sync — all while keeping your Premiumize bandwidth (Fair-Use points) as low as possible.

</div>

---

## ✨ Features

- **Stream, don't download.** Your Premiumize library is mounted as a FUSE virtual filesystem. Plex/Jellyfin/Emby read directly from it; bytes are pulled from the Premiumize CDN on demand and nothing is stored to disk.
- **Search everywhere.** Query your Premiumize cache and your Prowlarr indexers from one web UI. Cached results appear instantly; uncached ones are queued on Premiumize.
- **Instant auto-import.** Click *Add* and the content is registered with the matching Radarr/Sonarr instance and imported immediately — no waiting, no manual reconcile.
- **Stub-first scanning + a one-time Import button.** New content is first served as a tiny stub so Plex can scan your whole library at **zero CDN cost**. One click then flips everything to real streaming — permanently. (See [The Import button](#-the-import-button-important).)
- **4K-aware routing.** 4K/UHD content is automatically routed to your 4K Radarr/Sonarr instances and HD content to your HD instances, into separate `movies4k` / `series4k` mount folders.
- **Bulk library sync.** Full Sync or Partial Sync your entire Premiumize library into Radarr/Sonarr, with a live progress stream.
- **Bandwidth-aware streaming.** An analysis-aware prefetch gate keeps Plex's metadata scans cheap (a few MB per file instead of hundreds), a disk cache means you never fetch the same bytes twice, and chunk size / read-ahead are tunable live.
- **Video-files-only mode.** Optionally hide NFOs, samples, extras, and non-video files so libraries stay clean.
- **TMDB metadata.** Optional TMDB key generates `.nfo` metadata for accurate titles, posters, and episode data. Supports both the v3 API key and the v4 Read Access Token.
- **Self-healing FUSE mount.** Clean unmount on shutdown, automatic cleanup of stale mounts on restart, and a watchdog that detects and recovers a wedged mount without a full restart.
- **Self-contained backup & restore.** Export everything — your cached transfers *and* your config (including the Premiumize key) — in one file.

---

## 📋 Requirements

- **Docker** with Compose
- **Premiumize.me** account and API key — from [premiumize.me/account](https://premiumize.me/account)
- **Prowlarr** *(optional, recommended)* — for searching indexers
- **Radarr and/or Sonarr** *(optional)* — for library management and auto-import
- **Plex / Jellyfin / Emby** *(optional)* — for streaming
- A host kernel with FUSE support (`/dev/fuse`). The container runs privileged for the in-container mount.

---

## 🚀 Getting Started

### 1. Extract

```bash
mkdir -p /docker/premiumize-web
cd /docker/premiumize-web
tar -xzf premiumize-web.tar.gz --strip-components=1
```

### 2. Configure

Create a `.env` next to `docker-compose.yml`:

```env
PREMIUMIZE_API_KEY=your_api_key_here
TORZNAB_APIKEY=change-me        # Prowlarr uses this to authenticate
PROWLARR_URL=http://192.168.1.10:9696
PROWLARR_API_KEY=your_prowlarr_key
```

Only `PREMIUMIZE_API_KEY` is strictly required to start; everything else can also be set later in the web UI under **Settings**.

### 3. Start

```bash
docker compose up -d --build
```

Open **http://your-server-ip:5000**.

### 4. Connect your media stack

<details>
<summary><b>Prowlarr</b> — add premiumize-web as a Torznab indexer</summary>

Indexers → Add → Torznab:

| Field | Value |
|---|---|
| URL | `http://your-server-ip:5000/torznab` |
| API Key | your `TORZNAB_APIKEY` |
| Categories | Movies (2000), TV (5000) |
</details>

<details>
<summary><b>Radarr / Sonarr</b> — add premiumize-web as a qBittorrent download client</summary>

Download Client → Add → qBittorrent:

| Field | Value |
|---|---|
| Host | `your-server-ip` |
| Port | `5000` |
| URL Base | `/qbt` |
| Category | `radarr` (or `sonarr`) |
| Username / Password | *(leave blank)* |

Add a **Remote Path Mapping** where Remote = Local = `/docker/premiumize-web/downloads/radarr` (or `/sonarr`).
</details>

> **First time?** See [Adding the indexer & download client to Radarr/Sonarr](#-adding-the-indexer--download-client-to-radarrsonarr) for full step-by-step instructions, including adding the indexer directly to the Arrs without Prowlarr.

<details>
<summary><b>Plex / Jellyfin / Emby</b> — point libraries at the mount</summary>

| Library | Path |
|---|---|
| Movies | `/docker/premiumize-web/mnt/movies` |
| TV Shows | `/docker/premiumize-web/mnt/series` |
| Movies 4K *(optional)* | `/docker/premiumize-web/mnt/movies4k` |
| TV 4K *(optional)* | `/docker/premiumize-web/mnt/series4k` |
</details>

### 5. Add or sync your content

Either search-and-Add individual items in the **Search** tab, or use the **Sync** tab to bulk-import your existing Premiumize library into Radarr/Sonarr.

### 6. Press the Import button — once

After Plex has finished its first scan of everything you added, go to **Settings → Plex** and click **✅ Import Complete — Switch to CDN**. This is a one-time step that switches the whole library from cheap scanning stubs to real streaming. It's important enough to have its own section below.

---

## The Import button (important)

> **Settings → Plex → "✅ Import Complete — Switch to CDN"**

This button is the single most important concept to understand, because it's what keeps your Premiumize bandwidth low during setup.

### Why it exists

When Plex adds a library, it **scans every file** — reading each one to detect duration, codec, and resolution. If Plex read the *real* video for every title straight from the Premiumize CDN, that initial scan of a large library would burn a huge amount of Fair-Use bandwidth before you've watched anything.

To avoid that, premiumize-web serves each new file as a **tiny stub** (a few hundred KB, correct resolution) instead of the real video. Plex happily scans the stub — at essentially **zero CDN cost** — and imports the item. Your whole library can be scanned for free.

### What the button does

Once Plex has finished that initial scan/import, you press the button **once**. It sets a permanent, one-way latch:

- Every file — everything already in the library **and anything you add later** — now serves the **real video bytes** from the Premiumize CDN.
- New content added after this point is a real file immediately. **No stub, no second press.**
- The setting persists across restarts.

### How to use it

1. Add or sync all your content (Search tab and/or Sync tab).
2. Let Plex finish scanning — the library populates from stubs, fast and free.
3. Click **Import Complete — Switch to CDN**.
4. Done. From now on, playback streams real files, and any new additions are real files automatically.

The button shows **✅ Import Complete** once it's been pressed, and stays that way. The only thing that changes after the switch is that the stub→real size change makes Plex re-scan each file once — but that re-scan is kept cheap by the prefetch gate (it reads only the header, a few MB, not the whole movie).

> **In short:** stubs make the big initial scan free; the button flips you into real-streaming mode forever. Press it once, after your first import settles.

---

## 🔗 Adding the indexer & download client to Radarr/Sonarr

This is the detailed version of step 4. There are two pieces: an **indexer** (so the Arrs can search premiumize-web for releases) and a **download client** (the fake qBittorrent, so the Arrs can "grab" and import them). You need both.

> Throughout, use your server's **LAN IP** (e.g. `192.168.1.50`), not `localhost` or `127.0.0.1` — Radarr/Sonarr usually run in their own containers and can't reach premiumize-web on loopback.

### Part 1 — Add the indexer

premiumize-web exposes a standard **Torznab** indexer at `/torznab`. You can add it either through Prowlarr (recommended if you already run it) or directly into each Arr.

#### Option A — via Prowlarr 

1. Prowlarr → **Indexers** → **Add Indexer** → search for and choose **Generic Torznab**.
2. Fill in:

   | Field | Value |
   |---|---|
   | Name | `Premiumize` |
   | URL | `http://your-server-ip:5000/torznab` |
   | API Key | your `TORZNAB_APIKEY` |
   | Categories | Movies (2000), TV (5000) |

3. **Test**, then **Save**.
4. Prowlarr will push the indexer to Radarr/Sonarr automatically (make sure those are added under Prowlarr → Settings → Apps).

#### Option B — directly in Radarr/Sonarr (no Prowlarr) (recommended)

1. In Radarr (or Sonarr) → **Settings → Indexers** → **+** → choose **Torznab** (the "Custom" Torznab option).
2. Fill in:

   | Field | Value |
   |---|---|
   | Name | `Premiumize` |
   | URL | `http://your-server-ip:5000/torznab` |
   | API Path | `/api` *(default — resolves to `/torznab/api`)* |
   | API Key | your `TORZNAB_APIKEY` |
   | Categories | **Radarr:** Movies `2000` (and `2045` for UHD) · **Sonarr:** TV `5000` (and `5045` for UHD) |

3. Leave the rest at defaults. **Test** (it should report OK), then **Save**.

The indexer supports plain text search, movie search, and TV search with season/episode — so Radarr/Sonarr's automatic and interactive searches both work. Results that are already cached on Premiumize are flagged so they grab instantly.

### Part 2 — Configure the qBittorrent download client

premiumize-web pretends to be qBittorrent at `/qbt`. When an Arr "grabs" a release, it hands the magnet to this client, which writes the import stub and (optionally) auto-imports.

1. In Radarr (or Sonarr) → **Settings → Download Clients** → **+** → choose **qBittorrent**.
2. Fill in:

   | Field | Value | Notes |
   |---|---|---|
   | Name | `Premiumize` | anything |
   | Enable | ✅ on | |
   | Host | `your-server-ip` | LAN IP, **not** localhost |
   | Port | `5000` | same port as the web UI |
   | URL Base | `/qbt` | **required** — this is how the mock is reached |
   | Username | *(blank)* | no auth |
   | Password | *(blank)* | no auth |
   | Category | `radarr` *(Radarr)* / `sonarr` *(Sonarr)* | becomes the download subfolder |
   | Use SSL | ❌ off | |

3. **Test** — it should succeed (the client reports a qBittorrent version and accepts the empty login). Then **Save**.

> The **Category** you enter here is used as the subfolder under the downloads root: a category of `radarr` means stubs land in `/docker/premiumize-web/downloads/radarr`. Whatever you type must match the Remote Path Mapping below.

### Part 3 — Add a Remote Path Mapping

The download client tells the Arr the stub lives at `/docker/premiumize-web/downloads/<category>`. The Arr must be able to read that exact path, so add a mapping (even when the paths are identical — this is what makes import work reliably).

1. In Radarr/Sonarr → **Settings → Download Clients** → scroll to **Remote Path Mappings** → **+**.
2. Fill in:

   | Field | Value |
   |---|---|
   | Host | `your-server-ip` *(must exactly match the download client's Host)* |
   | Remote Path | `/docker/premiumize-web/downloads/radarr` *(or `/sonarr`)* |
   | Local Path | the same path **as the Arr sees it** |

3. **Save**.

If Radarr/Sonarr run in containers, make sure `/docker/premiumize-web/downloads` is mounted into them so the Local Path is actually reachable. When the Arr and premiumize-web share the same host bind, Remote Path and Local Path are identical.

### Verify

Search for a cached title in the Arr (interactive search), grab it, and within a few seconds it should show as **Downloaded** (or import from the queue). If it sits at "Downloading," wait one poll cycle (~60 s) or trigger a manual rescan. If the download client **Test** fails with *connection refused*, premiumize-web isn't reachable on `:5000` from the Arr — check the IP/port and that the container is running.

---

## The Web UI

### Search

Search your Premiumize cache and Prowlarr indexers. A green cloud means already cached (instant); a red cloud means it will be queued on Premiumize. Click **Add** to grab — if you've configured Arr instances in the Sync tab, it's also auto-imported into the right one.

Search syntax examples: `Oppenheimer 2023`, `Breaking Bad S03`, `The Bear Season 2`.

### Library

Browse everything currently in your FUSE mount; filter by name.

### Sync

Bulk-import your Premiumize library into Radarr/Sonarr.

**Configure instances** — one row each, with Type (Radarr/Sonarr), Quality (HD / 4K / **Any**), Name, URL, and API key. With separate HD and 4K instances, content routes by resolution automatically; with one of each set to **Any**, everything goes there.

**Choose a folder** to sync from: `movies`, `movies4k`, `series`, or `series4k`.

**Pick a mode:**

| Mode | What it does |
|---|---|
| **Full Sync** | Wipes existing stubs from the Arr root, sets all library items to monitored, then writes a fresh stub for every item in the mount and triggers a rescan. Best for a clean reset. |
| **Partial Sync** | Only adds stubs for mount items that don't already have one in the Arr root. Existing stubs are untouched. Fast — good for keeping up with new additions. |

Progress streams live, showing each title as it's processed.

### Settings

| Section | Purpose |
|---|---|
| **API Key** | Your Premiumize API key — required for everything. |
| **Prowlarr** | URL + key for search. |
| **Mount Configuration** | Live-tune FUSE streaming (chunk size, RAM buffer, read-ahead, video-only, timeouts) — applies immediately, no restart. |
| **Plex** | Plex URL/token for auto-scan, plus the **Import Complete** button. |
| **TMDB** | Optional metadata key (v3 key or v4 read token). Use **Test** to verify. |
| **💾 Debrid Backup & Restore** | Export/restore your cached transfers **and** config (incl. Premiumize key) in one file. |
| **Clear Downloads** | Removes the small import stubs from the downloads folder. Never touches real media. |

---

## Environment Variables

Set in `.env`. Most can also be changed live in the UI.

### Core

| Variable | Default | Description |
|---|---|---|
| `PREMIUMIZE_API_KEY` | *(required)* | Your Premiumize API key |
| `TORZNAB_APIKEY` | `changeme` | Key Prowlarr uses to authenticate |
| `ENABLE_FUSE` | `true` | Set `false` to disable the mount |
| `DOWNLOADS_ROOT` | `/docker/premiumize-web/downloads` | Where import stubs are written |
| `PORT` | `5000` | Web UI / API port |

### Prowlarr

| Variable | Description |
|---|---|
| `PROWLARR_URL` | e.g. `http://192.168.1.10:9696` |
| `PROWLARR_API_KEY` | Prowlarr → Settings → General |

### Plex auto-scan

| Variable | Default | Description |
|---|---|---|
| `PLEX_URL` | — | e.g. `http://192.168.1.10:32400` |
| `PLEX_TOKEN` | — | Your Plex token |
| `PLEX_MOVIES_SECTION` | `1` | Movies library section ID |
| `PLEX_TV_SECTION` | `2` | TV library section ID |
| `PLEX_SCAN_DEBOUNCE` | `30` | Seconds to batch bulk imports before scanning |

*Find your token:* play any item → ··· → Get Info → View XML → `X-Plex-Token=` in the URL.

### TMDB *(optional)*

| Variable | Description |
|---|---|
| `TMDB_API_KEY` | v3 API key (32-char) **or** v4 Read Access Token (JWT). Both are accepted. |

### FUSE streaming

| Variable | Default | Description |
|---|---|---|
| `FUSE_READAHEAD_MB` | `32` | Chunk size (MB). **Drop to `4` if movies stall or scans use too much bandwidth.** |
| `FUSE_CACHE_BLOCKS` | `64` | RAM blocks kept in memory (64 × chunk size). |
| `FUSE_READ_AHEAD` | `500MB` | How far ahead of playback to prefetch. |
| `FUSE_PREFETCH` | `true` | Master prefetch on/off. |
| `FUSE_PREFETCH_SEQ_THRESHOLD` | `2` | Consecutive blocks of sequential reads before prefetch engages. Keeps scans cheap; `0` disables the gate. |
| `FUSE_VIDEO_ONLY` | `true` | Show only video files — hides NFOs, samples, and extras. |
| `FUSE_READ_TIMEOUT` | `30` | CDN read timeout (seconds). |
| `FUSE_CACHE_DIR` | `/docker/premiumize-web/cache` | Disk cache location. |
| `FUSE_MOUNTPOINT` | `/docker/premiumize-web/mnt` | Mount location. |

---

## How it works

### The virtual filesystem

```
/docker/premiumize-web/mnt/
  movies/   The.Dark.Knight.2008.2160p.BluRay/  The.Dark.Knight.2008.2160p.BluRay.mkv
  series/   Breaking.Bad.S03E07.1080p/          Breaking.Bad.S03E07.1080p.mkv
```

Files are never downloaded. They stream from the Premiumize CDN in real time, and the disk cache stores blocks you've already fetched so seeking back is instant and the same bytes are never pulled twice. Movie vs. series is detected from the name (`S01E01`, `Season 1`, `1x01` → series); 4K vs. HD from the resolution token.

### Stubs and the Arr import trick

Radarr and Sonarr expect files on disk. premiumize-web acts as a qBittorrent client and writes small `.mkv` stubs (correct pixel dimensions so MediaInfo reports the right quality tier) into the downloads folder. Arr imports the stub and marks the item "Downloaded"; actual playback happens through the mount. Season packs get one stub per episode so every episode imports.

The same stubs are what the mount serves for scanning **until you press the Import button** — then the mount serves real CDN bytes (see above).

### Bandwidth control

The biggest cost isn't playback — it's Plex *touching* files during scans and analysis. premiumize-web minimizes this with: stub-first scanning (free initial scan), an analysis-aware prefetch gate (a metadata probe reads a few MB, not the whole file), a disk cache (no re-fetching), and targeted Plex scans instead of full sweeps. For best results, also disable Plex's whole-file tasks (preview thumbnails, loudness analysis, intro/credit detection) — those decode entire files and no server-side change can stop them.

### Self-healing mount

On shutdown the mount is cleanly unmounted; on restart any stale "Transport endpoint is not connected" mount is cleared before remounting; and a watchdog probes the live mount every 15s, forcing a remount if it ever goes unresponsive — all with time-bounded operations so recovery itself can never hang. Keep `restart: unless-stopped` in compose so the process itself also comes back after a crash.

---

## 🛠️ Troubleshooting

**Movies stall or scanning uses too much bandwidth** — Set Chunk Size to `4MB` in Mount Configuration, confirm the prefetch gate is on (default), and disable Plex's video preview thumbnails, loudness analysis, and intro/credit detection. A movie that hangs at the very start is the header read; one that hangs partway is usually a Plex whole-file analysis task.

**Content shows "Downloading" not "Downloaded" in Arr** — Wait for Arr's next poll (~60s) or trigger a manual rescan.

**"Permission denied" importing stubs** — `chmod -R a+rw /docker/premiumize-web/downloads` (container and Arr run as different users).

**Sonarr sync freezes partway with no logs** — Update to the current build; the sync now emits a heartbeat per item (the status line shows the in-flight title) and skips per-item TMDB lookups when Video Files Only is on.

**TMDB test fails with a correct key** — The current build accepts both the v3 key and the v4 Read Access Token. If it still fails, check the container can reach `api.themoviedb.org`.

**Export Config / backup** — The backup is now self-contained (transfers + config + Premiumize key). Treat the file as a secret. Restoring an older backup overwrites changed settings (uploaded values win).

**FUSE mount appears empty** — `docker logs premiumize-web | grep -i fuse`; confirm a valid Premiumize key and `/dev/fuse` + `SYS_ADMIN` access.

**"Connection refused :5000" in Sonarr** — The app was down (restart/redeploy/crash). Confirm it's listening (`curl http://host:5000/`) and that compose has `restart: unless-stopped`. Sonarr retries on the next RSS pass.

---

## 🗄️ Backup & Restore via API

```bash
# Self-contained backup (transfers + config + PM key)
curl http://your-server-ip:5000/api/backup -o backup.json

# Restore (accepts both new and legacy backups)
curl -X POST http://your-server-ip:5000/api/restore -F "file=@backup.json"
```

---

## 🗑️ Uninstalling

```bash
cd /docker/premiumize-web
docker compose down --rmi all
umount -l /docker/premiumize-web/mnt 2>/dev/null || true
rm -rf /docker/premiumize-web
```

---

<div align="center">
<sub>Built for people who want their media stack to just work — without burning bandwidth to do it.</sub>
</div>
