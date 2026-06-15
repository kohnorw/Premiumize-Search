# premiumize-web

Web UI + qBittorrent mock + FUSE virtual filesystem for Premiumize.me.

## First-time setup

Run once on the host as root before deploying:

```bash
sudo ./setup.sh
```

This creates the shared bind-mount that allows the FUSE filesystem to propagate
from inside the container to the host, and persists it across reboots via `/etc/rc.local`.

## Deploy

```bash
cd /docker/premiumize-web
cp .env.example .env        # add your PREMIUMIZE_API_KEY
docker compose up -d --build
```

## Mount structure

```
/docker/premiumize-web/mnt/
  movies/   ← auto-detected by name (no season code)
  series/   ← auto-detected by S01E01 / Season 1 patterns
  cache/    ← persistent block cache
```

Point Plex at `/docker/premiumize-web/mnt/movies` and `/docker/premiumize-web/mnt/series`.

## Sonarr / Radarr

Add as a qBittorrent download client:

| Setting       | Value                        |
|---------------|------------------------------|
| Host          | `<your-server-ip>`           |
| Port          | `5000`                       |
| URL Base      | `/qbt`                       |
| Category      | `tv-sonarr` / `radarr`       |
| Username/Pass | leave blank                  |

Remote path mapping:
- Radarr: `/docker/premiumize-web/mnt/movies`
- Sonarr: `/docker/premiumize-web/mnt/series`

## Uninstall

```bash
# 1. Stop and remove the container
cd /docker/premiumize-web
docker compose down --rmi all

# 2. Unmount any stale FUSE mounts
umount -l /docker/premiumize-web/mnt 2>/dev/null || true
umount -l /docker/premiumize-web 2>/dev/null || true

# 3. Remove the systemd unit (installed by setup.sh)
systemctl stop premiumize-web-rshared.service
systemctl disable premiumize-web-rshared.service
rm -f /etc/systemd/system/premiumize-web-rshared.service
systemctl daemon-reload

# 4. Delete all data
rm -rf /docker/premiumize-web
```

| Variable              | Default     | Description                        |
|-----------------------|-------------|------------------------------------|
| `PREMIUMIZE_API_KEY`  | —           | Required                           |
| `TORZNAB_APIKEY`      | `changeme`  | API key for Torznab endpoint       |
| `PROWLARR_URL`        | —           | Optional Prowlarr URL              |
| `PROWLARR_API_KEY`    | —           | Optional Prowlarr API key          |
| `ENABLE_FUSE`         | `true`      | Set to `false` to disable FUSE     |
