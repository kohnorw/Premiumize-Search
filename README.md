# premiumize-web

A self-hosted web UI and Torznab indexer for searching torrents and streaming cached content instantly via **Premiumize**. No files are stored in your Premiumize cloud — cached torrents are streamed directly from the CDN.

![Search UI](https://img.shields.io/badge/UI-Web%20%2B%20VLC-orange) ![Python](https://img.shields.io/badge/Python-3.10%2B-blue) ![Flask](https://img.shields.io/badge/Flask-3.0-green)

---

## Features

- **Search** torrents via Prowlarr (all your configured indexers) or TPB as a fallback
- **Cache check** — instantly see which results are available on Premiumize (no waiting)
- **▶ Play** cached results without adding them to your cloud:
  - `.mkv` and other formats open directly in **VLC** via the `vlc://` protocol
  - `.mp4` plays in the browser inline
  - Season packs are organized by season with collapsible groups and a file filter
- **Add** button to send a magnet to your Premiumize cloud transfers
- **Streaming proxy** — Flask proxies the Premiumize CDN stream so VLC gets a clean URL regardless of which CDN they use
- **Torznab endpoint** for Prowlarr/Sonarr/Radarr with proper rate limiting so indexers never get disabled
- **Settings tab** to configure Prowlarr from the UI — tests the connection and lists active indexers

---

## Quick Start

### Docker (recommended)

```bash
PREMIUMIZE_API_KEY=your_key docker compose up
```

### Without Docker

```bash
pip install -r requirements.txt
PREMIUMIZE_API_KEY=your_key python app.py
```

Open **http://localhost:5000**

Get your Premiumize API key at https://www.premiumize.me/account

---

## Environment Variables

| Variable               | Default      | Description                                                  |
|------------------------|--------------|--------------------------------------------------------------|
| `PREMIUMIZE_API_KEY`   | *(empty)*    | Pre-fills the Premiumize API key in the UI                   |
| `TORZNAB_APIKEY`       | `changeme`   | API key required by Prowlarr to use the Torznab endpoint     |
| `TORZNAB_RATE_LIMIT`   | `60`         | Max Torznab requests per window before returning 429         |
| `TORZNAB_RATE_WINDOW`  | `60`         | Rate limit window in seconds                                 |
| `CONFIG_PATH`          | `/app/config.json` | Where Prowlarr settings are persisted                  |
| `PORT`                 | `5000`       | Port the server listens on                                   |

---

## Prowlarr Setup

### Using this app as a Torznab indexer in Prowlarr

1. In Prowlarr go to **Indexers → Add → Generic Torznab**
2. Set the URL to `http://your-host:5000/torznab`
3. Set the API key to `changeme` (or your `TORZNAB_APIKEY` value)
4. Save — Prowlarr will auto-detect capabilities

The Torznab endpoint returns `X-RateLimit-*` headers on every response and a proper `429 Too Many Requests` with `Retry-After` when the limit is hit, so Prowlarr will back off instead of disabling the indexer.

### Using Prowlarr as a search source

1. Open the **⚙️ Settings** tab in the web UI
2. Enter your Prowlarr URL (e.g. `http://192.168.1.x:9696`) and API key
3. Click **Save & Test** — it will connect and list your active torrent indexers
4. Searches will now pull results from all your Prowlarr indexers instead of TPB

---

## Playing Cached Torrents

Clicking **▶ Play** on a cached result:

1. Calls Premiumize's `directdl` API to get direct CDN links — **nothing is added to your cloud**
2. Opens a file browser modal organized by season (for series) with a filter search box
3. Each file has:
   - **▶ VLC** — opens the file in your locally installed VLC via `vlc://` protocol
   - **▶ Browser** — plays `.mp4` files inline (not available for `.mkv` due to browser codec limitations)
   - **⬇ Download** — direct download link

The VLC stream goes through a local proxy (`/api/stream/<token>`) that strips the `Content-Disposition: attachment` header and forwards range requests for seeking. Stream tokens expire after 1 hour.

> **VLC requirement:** VLC must be installed and registered as the `vlc://` protocol handler. This is the default on Windows and macOS after a standard VLC install. On Linux you may need to register it manually.

---

## API Reference

| Method   | Path                        | Description                                          |
|----------|-----------------------------|------------------------------------------------------|
| `POST`   | `/api/search`               | Search torrents and check Premiumize cache           |
| `POST`   | `/api/add`                  | Add a magnet to Premiumize cloud                     |
| `POST`   | `/api/links`                | Get directdl links for a cached hash                 |
| `GET`    | `/api/stream/<token>`       | Proxy stream a CDN URL by token (range-request aware)|
| `POST`   | `/api/account`              | Verify Premiumize key and return premium status      |
| `GET`    | `/api/prowlarr/config`      | Get current Prowlarr configuration                   |
| `POST`   | `/api/prowlarr/config`      | Save and test Prowlarr configuration                 |
| `DELETE` | `/api/prowlarr/config`      | Remove Prowlarr configuration                        |
| `GET`    | `/api/prowlarr/indexers`    | List active Prowlarr torrent indexers                |
| `GET`    | `/torznab`                  | Torznab-compatible XML endpoint for Prowlarr         |

---

## How It Works

```
Browser / Prowlarr
      │
      ▼
  Flask App (port 5000)
      │
      ├── Search ──► Prowlarr /api/v1/search  (or TPB fallback)
      │                    │
      │                    ▼
      ├── Cache check ──► Premiumize /api/cache/check
      │                    (plain info hashes, parallel array response)
      │
      ├── Play ──► Premiumize /api/transfer/directdl
      │                    │
      │                    ▼
      │             CDN URL (energycdn.com or similar)
      │                    │
      │             /api/stream/<token>  ◄── VLC connects here
      │             (proxy, strips Content-Disposition, forwards Range)
      │
      └── Add ──► Premiumize /api/transfer/create
```

---

## Notes

- **Cached only for Play** — the ▶ Play button only appears on results Premiumize has cached. Non-cached results can still be Added to your cloud.
- **CDN domains vary** — Premiumize uses third-party CDNs (e.g. `energycdn.com`). The proxy accepts any `http/https` URL returned by `directdl` rather than allowlisting specific domains.
- **Season detection** parses `S01E04`, `S01`, and `Season 1` patterns from filenames. Files that don't match are grouped under "Other files".
- **Prowlarr config** is persisted to `config.json` and survives restarts.
