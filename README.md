# premiumize-web

A self-hosted web UI for searching torrents and streaming cached content instantly through [Premiumize](https://www.premiumize.me). Also exposes a Torznab endpoint so it works as an indexer in Prowlarr, Sonarr, and Radarr.

---

## How it works

1. Search for a movie or show
2. Results are instantly checked against the Premiumize cache
3. Cached results can be streamed directly — nothing is added to your Premiumize cloud
4. Series are organized by season with a filter box for easy browsing
5. Videos open in VLC on your machine via the `vlc://` protocol

---

## Quick start

### Docker

Create a `.env` file:

```env
PREMIUMIZE_API_KEY=your_premiumize_api_key
PROWLARR_URL=http://192.168.1.x:9696
PROWLARR_API_KEY=your_prowlarr_api_key
TORZNAB_APIKEY=changeme
```

Then run:

```bash
docker compose up -d
```

Open **http://localhost:5000**

### Without Docker

```bash
pip install -r requirements.txt
PREMIUMIZE_API_KEY=your_key python app.py
```

---

## Environment variables

| Variable              | Default              | Description                                              |
|-----------------------|----------------------|----------------------------------------------------------|
| `PREMIUMIZE_API_KEY`  | —                    | Premiumize API key. Found at premiumize.me/account       |
| `PROWLARR_URL`        | —                    | Prowlarr base URL e.g. `http://192.168.1.x:9696`         |
| `PROWLARR_API_KEY`    | —                    | Prowlarr API key. Found in Prowlarr → Settings → General |
| `TORZNAB_APIKEY`      | `changeme`           | Key Prowlarr uses to talk to this app's Torznab endpoint |
| `TORZNAB_RATE_LIMIT`  | `60`                 | Max requests to the Torznab endpoint per window          |
| `TORZNAB_RATE_WINDOW` | `60`                 | Rate limit window in seconds                             |
| `CONFIG_PATH`         | `/config/config.json`| Where Prowlarr settings are saved                        |
| `PORT`                | `5000`               | Port the server listens on                               |

If `PREMIUMIZE_API_KEY` or `PROWLARR_URL`/`PROWLARR_API_KEY` are set via environment, the UI won't ask for them. You can still override them in the Settings tab.

---

## Search sources

The app searches in this order:

1. **Prowlarr** — if configured, searches all your torrent indexers via `/api/v1/search`
2. **The Pirate Bay** — fallback if Prowlarr is not configured or returns no results

Configure Prowlarr in the **⚙️ Settings** tab or via environment variables.

---

## Playing cached torrents

Clicking **▶ Play** on a cached result:

- Calls Premiumize's `directdl` API to get direct CDN links — no transfer is created, nothing is saved to your cloud
- Opens a file browser organized by season for TV series, flat list for movies
- Use the **Filter files…** box to quickly find an episode
- **▶ VLC** streams the file in your locally installed VLC via the `vlc://` protocol handler
- **▶ Browser** plays `.mp4` files inline in the modal
- **⬇ Download** opens the direct CDN link

The stream goes through a local proxy (`/api/stream/<token>`) that strips the forced-download header from the CDN response and supports range requests so VLC can seek. Tokens expire after 1 hour.

> VLC must be installed. On Windows and macOS it registers the `vlc://` protocol handler automatically. On Linux you may need to register it manually.

The **Add** button sends the magnet to your Premiumize cloud transfers without streaming.

---

## Prowlarr integration

### Using this app as a Torznab indexer in Prowlarr

This lets Sonarr and Radarr search through this app.

1. In Prowlarr go to **Indexers → Add Indexer → Generic Torznab**
2. Set URL to `http://your-host:5000/torznab`
3. Set API key to whatever you set `TORZNAB_APIKEY` to (default: `changeme`)
4. Save and test

The endpoint returns proper `X-RateLimit-*` headers and `429 Too Many Requests` with `Retry-After` when the limit is hit, so Prowlarr backs off gracefully instead of disabling the indexer.

### Using Prowlarr as a search source in this app

1. Open the **⚙️ Settings** tab
2. Enter your Prowlarr URL and API key
3. Click **Save & Test** — it will verify the connection and list your active indexers
4. All searches will now pull from Prowlarr instead of TPB

---

## Settings tab

- **Prowlarr** — connect to your Prowlarr instance and see which indexers are active
- **Torznab endpoint** — shows the URL to paste into Prowlarr, plus current rate limit settings

---

## API endpoints

| Method   | Path                     | Description                                        |
|----------|--------------------------|----------------------------------------------------|
| GET      | `/`                      | Web UI                                             |
| GET      | `/torznab`               | Torznab endpoint for Prowlarr/Sonarr/Radarr        |
| GET      | `/api/config`            | Returns which credentials are set server-side      |
| POST     | `/api/search`            | Search torrents and check Premiumize cache         |
| POST     | `/api/add`               | Add a magnet to Premiumize cloud                   |
| POST     | `/api/links`             | Get directdl links for a cached hash               |
| GET      | `/api/stream/<token>`    | Proxy stream a CDN URL (range-aware)               |
| POST     | `/api/account`           | Verify Premiumize key and return account status    |
| GET      | `/api/prowlarr/config`   | Get current Prowlarr config                        |
| POST     | `/api/prowlarr/config`   | Save and test Prowlarr config                      |
| DELETE   | `/api/prowlarr/config`   | Remove saved Prowlarr config                       |
| GET      | `/api/prowlarr/indexers` | List active Prowlarr torrent indexers              |

---

## Requirements

- Python 3.10+
- VLC installed on the client machine for the ▶ VLC button
- A Premiumize premium account
- Prowlarr (optional, but recommended for better search results)
