<div align="center">

# 🎬 premiumize-web

**Stream your Premiumize library instantly through Plex, Radarr, and Sonarr**

premiumize-web is a self-hosted bridge that connects [Premiumize.me](https://premiumize.me) to your media stack. It surfaces your cached torrents as a virtual filesystem that Plex can stream directly — no downloading to disk, no waiting, just instant playback.

</div>

---

## ✨ What does it do?

When you search for a movie or TV show in Radarr or Sonarr, premiumize-web checks whether it already exists in the Premiumize cache. If it does, the file appears instantly in a virtual folder that Plex can play from — streamed directly from Premiumize's CDN. If it isn't cached yet, it gets added to your Premiumize download queue and appears automatically once it's ready.

```
You click "Search" in Radarr / Sonarr
         │
         ▼
  premiumize-web searches Prowlarr indexers
         │
         ▼
  Checks each result against Premiumize cache
         │
    ┌────┴────┐
    │         │
  Cached    Not cached
    │         │
    ▼         ▼
Appears    Added to Premiumize
instantly  download queue
in Plex
```

---

## 📋 Requirements

Before you start, make sure you have:

- **Docker** with Compose (any recent version)
- **A Premiumize.me account** — you need your API key from [premiumize.me/account](https://premiumize.me/account)
- **Prowlarr** — for searching torrent indexers *(optional but strongly recommended)*
- **Radarr and/or Sonarr** — for managing your media library *(optional)*
- **Plex** — for streaming *(optional)*

---

## 🚀 Quick Start

### Step 1 — Get the files

```bash
mkdir -p /docker/premiumize-web
cd /docker/premiumize-web
# Extract the downloaded tar here
tar -xzf premiumize-web.tar.gz --strip-components=1
```

### Step 2 — Create your config file

```bash
cp .env.example .env
nano .env
```

At minimum, fill in your Premiumize API key:

```env
PREMIUMIZE_API_KEY=your_api_key_here
```

### Step 3 — Start it up

```bash
docker compose up -d --build
```

Open **http://your-server-ip:5000** — you should see the web UI.

---

## ⚙️ Configuration

### Environment variables (`.env` file)

These are set once in your `.env` file and apply at startup. The web UI can override most of them at runtime.

---

#### 🔑 Core Settings

| Variable | Required | Default | Description |
|---|---|---|---|
| `PREMIUMIZE_API_KEY` | ✅ Yes | — | Your Premiumize API key. Get it from [premiumize.me/account](https://premiumize.me/account) |
| `TORZNAB_APIKEY` | — | `changeme` | The password Prowlarr uses to talk to premiumize-web. Change this to something unique. |
| `ENABLE_FUSE` | — | `true` | Set to `false` to disable the virtual filesystem (useful for testing without FUSE) |

---

#### 🔍 Prowlarr Integration

Prowlarr is what premiumize-web uses to search torrent indexers. Without it, only your existing Premiumize library is visible.

| Variable | Default | Description |
|---|---|---|
| `PROWLARR_URL` | — | URL to your Prowlarr instance, e.g. `http://192.168.1.10:9696` |
| `PROWLARR_API_KEY` | — | Your Prowlarr API key (found in Prowlarr → Settings → General) |
| `TORZNAB_RATE_LIMIT` | `60` | Max searches per rate window (prevents hammering Prowlarr) |
| `TORZNAB_RATE_WINDOW` | `60` | Rate window in seconds |

These can also be set in the web UI under **Settings → Prowlarr**.

---

#### 📺 Plex Auto-Scan

When a file becomes available, premiumize-web can automatically tell Plex to scan its library — so new content appears in Plex within seconds rather than waiting for the next scheduled scan.

| Variable | Default | Description |
|---|---|---|
| `PLEX_URL` | — | URL to your Plex server, e.g. `http://192.168.1.10:32400` |
| `PLEX_TOKEN` | — | Your Plex authentication token (see below) |
| `PLEX_MOVIES_SECTION` | `1` | Plex library section ID for Movies |
| `PLEX_TV_SECTION` | `2` | Plex library section ID for TV Shows |
| `PLEX_SCAN_DEBOUNCE` | `30` | Seconds to wait after the last new file before triggering a scan. Prevents a bulk import from firing 50 individual scans — they're all batched into one. |

**Finding your Plex token:**
1. Open Plex and play any media item
2. Click ··· → Get Info → View XML
3. The token is in the URL: `?X-Plex-Token=`**`YOUR_TOKEN`**

**Finding your section IDs:**
Go to Plex → Settings → Libraries. Each library has an ID visible in its URL when you click on it (e.g. `/library/sections/`**`1`**).

---

#### 💾 FUSE Streaming Settings

These control how the virtual filesystem buffers and caches data from the Premiumize CDN. They can all be set in the web UI under **Mount Configuration**, or via environment variables.

| Variable | Default | Description |
|---|---|---|
| `FUSE_READAHEAD_MB` | `32` | Size of each data chunk in MB. Larger = fewer round trips to the CDN per second of video. |
| `FUSE_CACHE_BLOCKS` | `64` | Number of chunks to keep in RAM. `64 × 32MB = 2GB` of in-memory buffer. Reduce if your server has limited RAM. |
| `FUSE_READ_AHEAD` | `512MB` | How far ahead of the playback position to pre-fetch. `512MB ÷ 32MB = 16 chunks` of runway. |
| `FUSE_CACHE_DIR` | `/docker/premiumize-web/cache` | Where to store the persistent disk cache. |
| `FUSE_PREFETCH` | `true` | Enable background pre-fetching. Keeps chunks loaded ahead of where you're watching. |
| `FUSE_VIDEO_ONLY` | `true` | Only show video files in the virtual filesystem. Hides NFOs, samples, subtitles, and extras. |
| `FUSE_READ_TIMEOUT` | `30` | Seconds to wait for the Premiumize CDN before retrying. Increase this if you get frequent stutters on 4K content. |

**Recommended settings for smooth playback:**

| Content | Chunk Size | Buffer Memory | Read Ahead |
|---|---|---|---|
| 1080p | 16MB | 1GB | 256MB |
| 4K HDR | 32MB | 2GB | 512MB |
| 4K with slow CDN | 32MB | 4GB | 1GB |

---

## 🔌 Connecting your media stack

### Prowlarr

Add premiumize-web as a Torznab indexer in Prowlarr so Radarr and Sonarr can search through it.

1. Go to **Prowlarr → Indexers → Add Indexer**
2. Search for **Torznab** and select it
3. Fill in:

| Field | Value |
|---|---|
| Name | `Premiumize` |
| URL | `http://your-server-ip:5000/torznab` |
| API Key | Your `TORZNAB_APIKEY` value |
| Categories | Movies (2000), TV (5000) |

4. Click **Test** then **Save**

---

### Radarr

**Download Client:**
1. Go to **Settings → Download Clients → Add → qBittorrent**
2. Fill in:

| Field | Value |
|---|---|
| Name | `Premiumize` |
| Host | `your-server-ip` |
| Port | `5000` |
| URL Base | `/qbt` |
| Category | `radarr` |
| Username | *(leave blank)* |
| Password | *(leave blank)* |

3. Click **Test** then **Save**

**Remote Path Mapping:**
Go to **Settings → Download Clients → Remote Path Mappings → Add**:

| Field | Value |
|---|---|
| Host | `your-server-ip` |
| Remote Path | `/docker/premiumize-web/mnt/movies` |
| Local Path | `/docker/premiumize-web/mnt/movies` |

---

### Sonarr

**Download Client:** Same as Radarr above, but set Category to `tv-sonarr`.

**Remote Path Mapping:**

| Field | Value |
|---|---|
| Host | `your-server-ip` |
| Remote Path | `/docker/premiumize-web/mnt/series` |
| Local Path | `/docker/premiumize-web/mnt/series` |

---

### Plex

Add two libraries pointing at the virtual filesystem:

1. **Movies** → Add Library → Movies → Add Folder → `/docker/premiumize-web/mnt/movies`
2. **TV Shows** → Add Library → TV Shows → Add Folder → `/docker/premiumize-web/mnt/series`

Once configured with the Plex auto-scan variables above, new content will appear in Plex automatically within about 30 seconds of becoming available.

---

## 🗂️ How the virtual filesystem works

The FUSE mount lives at `/docker/premiumize-web/mnt/` and organises your content into two folders automatically:

```
/docker/premiumize-web/mnt/
  movies/
    The.Dark.Knight.2008.2160p/
      The.Dark.Knight.2008.2160p.mkv
  series/
    Breaking.Bad.S01/
      Breaking.Bad.S01E01.mkv
      Breaking.Bad.S01E02.mkv
  cache/   ← managed automatically, don't touch
```

**Movies vs Series detection** is based on the torrent name. Anything with a pattern like `S01E01`, `Season 1`, or `1x01` goes into `series/`. Everything else goes into `movies/`.

Files are **never downloaded to disk** — they stream from the Premiumize CDN in real time, with blocks cached locally to make seeking fast and avoid re-fetching the same data.

---

## 🌐 Web UI

The web UI at **http://your-server-ip:5000** gives you:

- **Search** — search your Premiumize library and Prowlarr indexers, see at a glance which results are already cached, and add them with one click
- **Library** — browse everything currently available in your virtual filesystem
- **Settings** — configure your Premiumize key, Prowlarr connection, and all streaming options without editing files
- **Mount Configuration** — live tuning of all FUSE streaming parameters

---

## 🛠️ Troubleshooting

**No results when searching in Radarr / Sonarr**
- Check that Prowlarr is reachable: open `http://your-prowlarr-ip:9696` in a browser
- Check the Prowlarr URL in Settings — from inside Docker, use the host's LAN IP (e.g. `192.168.1.x`) not `localhost`
- Confirm your Premiumize API key is set — without it, cache checking is skipped

**Red cloud icon in Radarr / Sonarr when downloading**
- Confirm the download client host/port points to premiumize-web (`port: 5000`, `url base: /qbt`)
- Run `docker logs premiumize-web --tail=30` at the moment you click download

**Buffering or stuttering in Plex**
- Increase Chunk Size to `32MB` and Buffer Memory to `2GB` in Mount Configuration
- If 4K specifically is stuttering, increase Daemon Timeout to `30s`
- Check `docker logs premiumize-web | grep "Block fetch"` for CDN errors

**"Unexpected failure before end of file" in Plex**
- This means a CDN block fetch timed out — increase `FUSE_READ_TIMEOUT` to `30` or higher in your `.env`
- Also check that your container is running the latest version of `fuse_mount.py`

**Content not appearing in Plex**
- Trigger a manual library scan in Plex first to confirm the file is visible
- Check that Plex has read access to `/docker/premiumize-web/mnt/`
- If using Plex auto-scan, double-check `PLEX_URL`, `PLEX_TOKEN`, and section IDs in your `.env`

**FUSE mount shows as empty**
- Run `docker logs premiumize-web | grep -i fuse` to check mount status
- Make sure `PREMIUMIZE_API_KEY` is set — the mount appears but shows no files without it

---

## 🗄️ Backup and restore

Export your entire transfer database:

```bash
curl http://your-server-ip:5000/api/backup -o premiumize-backup.json
```

Restore it (on the same or a new install):

```bash
curl -X POST http://your-server-ip:5000/api/restore \
  -F "file=@premiumize-backup.json"
```

Only hashes currently cached on Premiumize are imported — anything that has expired is skipped automatically.

---

## 🗑️ Uninstalling

```bash
cd /docker/premiumize-web
docker compose down --rmi all
umount -l /docker/premiumize-web/mnt 2>/dev/null || true
rm -rf /docker/premiumize-web
```

