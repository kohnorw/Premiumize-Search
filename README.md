<div align="center">

# 🎬 premiumize-web

**Stream your Premiumize library instantly. Search, grab, and auto-import into Radarr and Sonarr.**

premiumize-web is a self-hosted bridge between [Premiumize.me](https://premiumize.me) and your media stack. It mounts your Premiumize cloud library as a virtual filesystem, fakes a qBittorrent download client so Radarr and Sonarr can import files instantly, and keeps everything in sync automatically.

</div>

---

## ✨ What it does

- **Searches** your Premiumize cache and Prowlarr indexers from a single web UI
- **Grabs** cached torrents instantly — no downloading to disk
- **Mounts** your Premiumize library as a FUSE virtual filesystem that Plex, Jellyfin, or Emby can stream from directly
- **Auto-imports** grabbed content into Radarr or Sonarr immediately after you click Add
- **Syncs** your entire Premiumize library into Radarr/Sonarr in bulk (Full Sync, Partial Sync, or Reconcile)
- **Routes** content to the right Arr instance automatically — 4K content goes to your 4K instance, HD to your HD instance

---

## 📋 Requirements

- **Docker** with Compose
- **Premiumize.me** account — get your API key from [premiumize.me/account](https://premiumize.me/account)
- **Prowlarr** *(optional but recommended)* — for searching torrent indexers
- **Radarr and/or Sonarr** *(optional)* — for library management and automatic import
- **Plex / Jellyfin / Emby** *(optional)* — for streaming

---

## 🚀 Quick Start

### 1. Extract the files

```bash
mkdir -p /docker/premiumize-web
cd /docker/premiumize-web
tar -xzf premiumize-web.tar.gz --strip-components=1
```

### 2. Configure your environment

```bash
cp .env.example .env
nano .env
```

At minimum you need your Premiumize API key:

```env
PREMIUMIZE_API_KEY=your_api_key_here
TORZNAB_APIKEY=changeme   # change this — Prowlarr uses it to authenticate
```

### 3. Start

```bash
docker compose up -d --build
```

Open **http://your-server-ip:5000**

---

## ⚙️ Environment Variables

Set these in your `.env` file. Most can also be changed live in the web UI under Settings.

### Core

| Variable | Default | Description |
|---|---|---|
| `PREMIUMIZE_API_KEY` | *(required)* | Your Premiumize API key |
| `TORZNAB_APIKEY` | `changeme` | API key Prowlarr uses to authenticate with premiumize-web |
| `ENABLE_FUSE` | `true` | Set to `false` to disable the virtual filesystem |
| `DOWNLOADS_ROOT` | `/docker/premiumize-web/downloads` | Where fake import stubs are written |

### Prowlarr

| Variable | Default | Description |
|---|---|---|
| `PROWLARR_URL` | — | e.g. `http://192.168.1.10:9696` |
| `PROWLARR_API_KEY` | — | Found in Prowlarr → Settings → General |

### Plex Auto-Scan

| Variable | Default | Description |
|---|---|---|
| `PLEX_URL` | — | e.g. `http://192.168.1.10:32400` |
| `PLEX_TOKEN` | — | Your Plex token (see below) |
| `PLEX_MOVIES_SECTION` | `1` | Plex library section ID for Movies |
| `PLEX_TV_SECTION` | `2` | Plex library section ID for TV Shows |
| `PLEX_SCAN_DEBOUNCE` | `30` | Seconds to wait before triggering a scan — batches bulk imports |

**Finding your Plex token:** Play any item → ··· → Get Info → View XML → look for `X-Plex-Token=` in the URL.

### FUSE Streaming

| Variable | Default | Description |
|---|---|---|
| `FUSE_READAHEAD_MB` | `32` | Chunk size in MB. Larger = fewer CDN round trips |
| `FUSE_CACHE_BLOCKS` | `64` | RAM blocks to keep. `64 × 32MB = 2GB` in memory |
| `FUSE_READ_AHEAD` | `512MB` | How far ahead of playback to prefetch |
| `FUSE_CACHE_DIR` | `/docker/premiumize-web/cache` | Disk cache location |
| `FUSE_VIDEO_ONLY` | `true` | Only show video files in the mount |
| `FUSE_READ_TIMEOUT` | `30` | CDN timeout in seconds |

**Recommended streaming settings:**

| Content | Chunk Size | RAM Buffer |
|---|---|---|
| 1080p | 16MB | 1GB |
| 4K HDR | 32MB | 2GB |
| 4K with slow CDN | 32MB | 4GB+ |

---

## 🔌 Connecting your media stack

### Prowlarr

Add premiumize-web as a Torznab indexer so Radarr and Sonarr can search through it:

1. Prowlarr → Indexers → Add Indexer → Torznab
2. Fill in:

| Field | Value |
|---|---|
| Name | `Premiumize` |
| URL | `http://your-server-ip:5000/torznab` |
| API Key | Your `TORZNAB_APIKEY` |
| Categories | Movies (2000), TV (5000) |

### Radarr

**Download Client** → Add → qBittorrent:

| Field | Value |
|---|---|
| Host | `your-server-ip` |
| Port | `5000` |
| URL Base | `/qbt` |
| Category | `radarr` |
| Username / Password | *(leave blank)* |

**Remote Path Mapping** → Add:

| Field | Value |
|---|---|
| Host | `your-server-ip` |
| Remote Path | `/docker/premiumize-web/downloads/radarr` |
| Local Path | `/docker/premiumize-web/downloads/radarr` |

### Sonarr

Same as Radarr, but set:
- Category: `sonarr`
- Remote Path: `/docker/premiumize-web/downloads/sonarr`
- Local Path: `/docker/premiumize-web/downloads/sonarr`

### Plex / Jellyfin / Emby

Add two libraries pointing at the virtual filesystem:

- **Movies** → `/docker/premiumize-web/mnt/movies`
- **TV Shows** → `/docker/premiumize-web/mnt/series`

New content appears automatically in Plex within ~30 seconds when auto-scan is configured.

---

## 🖥️ Web UI

Open **http://your-server-ip:5000** to access the web interface.

### Search Tab

Search your Premiumize cache and connected Prowlarr indexers. Each result shows:

- ✅ **Green cloud** — already cached on Premiumize, will appear instantly
- ❌ **Red cloud** — not cached, will be added to the Premiumize download queue

Click **Add** to grab a result. If you have Radarr/Sonarr instances configured in the Sync tab, the content is also automatically added to the right Arr instance and imported immediately — no manual reconcile needed.

**Search syntax:**
- `Breaking Bad S03` — season search
- `Oppenheimer 2023` — movie search
- `The Bear Season 2` — alternate season format

### Library Tab

Browse everything currently available in your FUSE mount. Filter by name to find specific content.

### Sync Tab

Bulk-import your Premiumize library into Radarr and/or Sonarr.

#### Configuring Instances

Add one row per Arr instance. Each instance has:

- **Type** — Radarr or Sonarr
- **Quality** — HD, 4K, or **Any**
  - `Any` — this instance receives both HD and 4K content
  - `HD` — only receives non-4K content
  - `4K` — only receives 2160p/UHD content
- **Name** — a label shown in the UI
- **URL** — e.g. `http://192.168.1.10:7878`
- **API Key** — found in your Arr instance under Settings → General

Example setup for a full 4K-aware stack:

| Type | Quality | Name | URL |
|---|---|---|---|
| Radarr | HD | Radarr | http://192.168.1.10:7878 |
| Radarr | 4K | Radarr 4K | http://192.168.1.10:7879 |
| Sonarr | HD | Sonarr | http://192.168.1.10:8989 |
| Sonarr | 4K | Sonarr 4K | http://192.168.1.10:8990 |

If you only have one Radarr and one Sonarr, set both to **Any** and they'll receive everything.

#### Sync Modes

**Full Sync**
Clears all existing episode/movie files in the Arr library, then imports everything found in the selected FUSE folder. For Sonarr, this does a per-episode match — only episodes present in the FUSE mount get fake files created. All series are set to monitored so Sonarr searches for any missing episodes.

Use this when starting fresh or after adding a large batch of content to Premiumize.

**Partial Sync**
Keeps your existing library intact. For Radarr, only adds movies that aren't already in the library. For Sonarr, only creates fake files for series already in your Sonarr library — it will not add new series. Use this for incremental updates.

**Reconcile**
Drives off your existing Arr library rather than the FUSE folder. For each movie or episode already in Radarr/Sonarr:
- If it's in the FUSE mount → create a fake file so Arr imports it
- If it's not in the FUSE mount → set it to monitored so Arr searches for it

Use this to fix up a library where some items show as missing when the files actually exist on Premiumize.

#### FUSE Folder

Choose which subfolder of the FUSE mount to scan:

| Folder | Contents |
|---|---|
| `movies` | Standard HD movies |
| `movies4k` | 4K/UHD movies |
| `series` | Standard HD TV |
| `tv4k` | 4K/UHD TV |

---

## 🔄 How auto-import works

When you click **Add** in the search UI:

1. The torrent is added to your Premiumize account (or marked as cached if it already is)
2. It appears immediately in the FUSE virtual filesystem
3. In the background, premiumize-web:
   - Detects whether it's a movie or series (from the title pattern)
   - Detects whether it's 4K or HD (from the resolution token in the name)
   - Picks the matching Arr instance based on type + quality
   - Adds the movie/series to that Arr instance if it isn't already there
   - Creates a small stub file in the downloads folder
   - Sends a `RescanMovie` or `RescanSeries` command to Arr
   - Arr imports the stub, marking the content as "Downloaded"
4. Plex auto-scan fires and the content appears in your library

The whole process typically completes within 5–10 seconds for cached content.

---

## 🗂️ How the virtual filesystem works

```
/docker/premiumize-web/mnt/
  movies/
    The.Dark.Knight.2008.2160p.BluRay/
      The.Dark.Knight.2008.2160p.BluRay.mkv
  series/
    Breaking.Bad.S03E07.1080p/
      Breaking.Bad.S03E07.1080p.mkv
```

Files are **never downloaded to disk** — they stream from the Premiumize CDN in real time. The local cache (`FUSE_CACHE_DIR`) stores blocks you've already fetched so seeking backward is instant and you never re-download the same data twice.

**Movies vs series** is detected automatically from the torrent name. Anything matching `S01E01`, `Season 1`, `1x01`, etc. goes into `series/`. Everything else goes into `movies/`.

---

## 🎭 How Arr importing works (the fake file trick)

Radarr and Sonarr don't know about cloud storage — they expect files on disk. premiumize-web works around this by acting as a qBittorrent download client and creating small stub `.mkv` files (a few hundred KB each) in the downloads folder. These stubs:

- Are real video files at the correct resolution (1080p stubs have actual 1920×1080 pixel dimensions, 4K stubs are 3840×2160)
- Pass MediaInfo quality detection so Radarr/Sonarr report the right quality tier
- Are created with open permissions so Radarr/Sonarr can move them

When Arr imports the stub, it moves it to your movies/series folder and marks the item as "Downloaded". The actual streaming happens via the FUSE mount — Plex reads from there, not from the imported stub.

For **season packs**, premiumize-web creates one stub per episode (e.g. `Show.S01E01.WEBRip-1080p.mkv`, `Show.S01E02.WEBRip-1080p.mkv`, etc.) so Sonarr can import all episodes individually.

---

## ⚙️ Settings Tab

### Prowlarr

Set your Prowlarr URL and API key. Used for searching when you type in the Search tab.

### API Key

Your Premiumize API key. Required for everything.

### Mount Configuration

Live-tune all FUSE streaming parameters without restarting. Changes apply immediately.

### Backup & Restore

Export your transfer database as JSON. Useful before updates or migrations. The restore function re-adds all transfers that are still cached on Premiumize — anything that has expired is skipped.

### Clear Downloads

Removes all stub files from `/docker/premiumize-web/downloads`. Safe to run at any time — these are only the small import stubs, not your actual media. Useful after a messy sync or if you want to force Arr to re-import everything.

---

## 🛠️ Troubleshooting

### No results when searching in Radarr/Sonarr
- Confirm Prowlarr is reachable and the indexer is added correctly
- Use the host's LAN IP inside Docker (e.g. `192.168.1.x`), not `localhost`
- Check that your Premiumize API key is set in Settings

### Content shows as "Downloading" in Arr instead of "Downloaded"
- The fake file was created but hasn't been imported yet — wait 60 seconds for Arr's next poll
- Or trigger a manual rescan in Arr → Movies/Series → Edit → Rescan

### "Permission denied" importing fake files
- Run `chmod -R a+rw /docker/premiumize-web/downloads` on the host
- This happens when the container and Arr run as different users

### Season pack only imports one episode
- Make sure you're on the latest version — older versions created one stub for the whole pack
- Current versions create one stub per episode so all episodes import correctly

### Sonarr series not getting set to monitored after Full Sync
- This is fixed in the current version — Full Sync now re-monitors all series after the reconcile pass

### FUSE mount appears empty
- Check `docker logs premiumize-web | grep -i fuse`
- Confirm your Premiumize API key is valid and your account has cached content
- Make sure the container has `SYS_ADMIN` capability and `/dev/fuse` device access (see `docker-compose.yml`)

### Buffering or stuttering in Plex
- Increase Chunk Size to `32MB` and Buffer Memory to `2GB` in Mount Configuration
- For 4K specifically, increase Read Timeout to `30s`
- Check `docker logs premiumize-web | grep "Block fetch"` for CDN errors

### "Episode file already imported" errors in Sonarr
- This is handled automatically — the reconcile skips episodes that already have files
- If it persists, run Clear Downloads in Settings and then a fresh Reconcile

---

## 🗄️ Backup and Restore

From the web UI → Settings → Backup & Restore, or via API:

```bash
# Download backup
curl http://your-server-ip:5000/api/backup -o backup.json

# Restore
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
<sub>Built for people who want their media stack to just work.</sub>
</div>
