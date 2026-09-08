# Tools configuration

This document describes the expected configuration of every service in the stack, in the order to set them up on first start, followed by recurring procedures. The values below match the `docker-compose.yml` of this repo: if you change a mount in the compose file, update this document.

Contents:

1. [Folder layout and paths](#1-folder-layout-and-paths)
2. [Setup order](#2-setup-order)
3. [Recurring procedures](#3-recurring-procedures)
4. [Cleanup after migrating from the old stack](#4-cleanup-after-migrating-from-the-old-stack)

---

## 1. Folder layout and paths

### 1.1 On the host

Two roots, defined in `.env`:

- `PATH_MEDIA`: the data (downloads, movies, TV shows). A dedicated, large disk.
- `PATH_CONF`: the containers' configuration. Small, to be backed up.

```
${PATH_MEDIA}/
├── downloads/              # qBittorrent: completed torrents
│   └── incomplete/         # qBittorrent: torrents in progress
├── complete/
│   ├── movies/             # Radarr root folder -> Plex library "Movies"
│   └── tv/                 # Sonarr root folder -> Plex library "TV Shows"
└── joal/
    └── conf/               # JOAL (separate stack, see joal/)

${PATH_CONF}/config/
├── homepage/
├── plex/
├── prowlarr/
├── qbittorrent/
├── radarr/
├── sonarr/
└── wireguard/
```

Initial creation, owned by the `PUID:PGID` user from `.env`:

```shell
source .env
mkdir -p "$PATH_MEDIA"/{downloads/incomplete,complete/movies,complete/tv,joal/conf/torrents}
mkdir -p "$PATH_CONF"/config/{homepage,plex,prowlarr,qbittorrent,radarr,sonarr,wireguard}
sudo chown -R "$PUID:$PGID" "$PATH_MEDIA" "$PATH_CONF"
```

### 1.2 As seen from each container

| Service     | Host                       | Container          | Role                                        |
|-------------|----------------------------|--------------------|---------------------------------------------|
| qBittorrent | `${PATH_MEDIA}`            | `/data`            | writes to `/data/downloads`                 |
| Sonarr      | `${PATH_MEDIA}`            | `/data`            | reads `/data/downloads`, files into `/data/complete/tv` |
| Radarr      | `${PATH_MEDIA}`            | `/data`            | reads `/data/downloads`, files into `/data/complete/movies` |
| Plex        | `${PATH_MEDIA}`            | `/media`           | reads `/media/complete/*`                   |
| Homepage    | `${PATH_MEDIA}`            | `/mnt/Zerepflix`   | disk usage widget only                      |
| All         | `${PATH_CONF}/config/<x>`  | `/config`          | persistent configuration                    |

Rule of thumb: **qBittorrent, Sonarr and Radarr see the same folder at the same path (`/data`)**. Consequences:

- No "Remote Path Mapping" to declare in Sonarr or Radarr.
- Sonarr and Radarr can **hardlink** instead of copying: a downloaded file keeps seeding from `downloads/` and shows up in `complete/` without using disk space twice. This only works if `downloads/` and `complete/` are on the same filesystem, so on the same disk.
- Plex sees the disk under a different name (`/media`) but never exchanges paths with the other services, so it does not matter.

---

## 2. Setup order

In this order, each service has what it needs by the time you reach it: qBittorrent → Prowlarr → Sonarr → Radarr → Plex → Homepage → WireGuard.

All URLs below use `localhost`, so they assume a browser running on the server itself. From another machine on the LAN, replace `localhost` with the server's IP or hostname.

### 2.1 qBittorrent

URL: `http://localhost:8081`

1. **First login.** The password is temporary and printed in the logs:
   ```shell
   docker logs qbittorrent 2>&1 | grep -i password
   ```
   Username `admin`. Change it right away: `Tools > Options > Web UI > Authentication`.
2. **Paths** (`Options > Downloads`):
   - Default Save Path: `/data/downloads`
   - Keep incomplete torrents in: `/data/downloads/incomplete` (checked)
   - **Never** leave `/downloads`: that path is not mounted, files end up inside the container and vanish on the next `docker compose up`.
3. **Categories.** Sonarr and Radarr create theirs automatically on first use (`tv-sonarr` and `radarr`). You can pre-create them in the sidebar, right click > Add category, leaving the path empty to inherit `/data/downloads`.
4. **Connection** (`Options > Connection`): listening port `6881`, same as the ports published in the compose file. UPnP disabled, the port is published manually.
5. **Web UI**: port `8081` must stay equal to `WEBUI_PORT` in the compose file. Sonarr and Radarr connect to it from the Docker network with the username and password set in step 1.

### 2.2 Prowlarr

URL: `http://localhost:9696`

Prowlarr centralizes indexers and pushes them to Sonarr and Radarr. **Never** add an indexer directly in Sonarr or Radarr.

1. **Authentication** (`Settings > General > Security`): method `Forms (Login Page)`, required `Enabled`. Prowlarr may start in `External` mode, meaning no login at all: fix it.
2. **Logging** (`Settings > General > Logging`): level `Info`. The `Debug` level fills `${PATH_CONF}/config/prowlarr/logs` with dozens of files.
3. **Tags** (`Settings > Tags`): create `sonarr` and `radarr`. They select which application each indexer is synced to.
4. **Indexers** (`Indexers > Add Indexer`): add your trackers, then tick the `sonarr` and/or `radarr` tags on each one. An indexer behind Cloudflare requires FlareSolverr, which is no longer in the stack: it will stay in error, disable it.
5. **Applications** (`Settings > Apps > Add`):

   | Field                | Sonarr                  | Radarr                  |
   |----------------------|-------------------------|-------------------------|
   | Sync Level           | Full Sync               | Full Sync               |
   | Tags                 | `sonarr`                | `radarr`                |
   | Prowlarr Server      | `http://prowlarr:9696`  | `http://prowlarr:9696`  |
   | Sonarr/Radarr Server | `http://sonarr:8989`    | `http://radarr:7878`    |
   | API Key              | see [3.1](#31-find-an-api-key-sonarr-radarr-prowlarr) | same |

   Use the **container names**, not `localhost` nor the LAN IP: containers talk to each other over the compose Docker network.
6. **Download Clients**: nothing to declare. Prowlarr only needs one for its own manual searches.

### 2.3 Sonarr

URL: `http://localhost:8989`

1. **Authentication** (`Settings > General > Security`): `Forms`, `Enabled`. Logging at `Info`.
2. **Root folder** (`Settings > Media Management > Root Folders`): `/data/complete/tv`.
3. **Importing** (`Settings > Media Management > Importing`, advanced settings): tick `Use Hardlinks instead of Copy`. Without it, every episode is duplicated between `downloads/` and `complete/`.
4. **Naming** (`Settings > Media Management > Episode Naming`): `Rename Episodes` checked. Formats in place:
   - Standard episode: `{Series Title} - S{season:00}E{episode:00} - {Episode Title} {Quality Full}`
   - Daily episode: `{Series Title} - {Air-Date} - {Episode Title} {Quality Full}`
   - Season folder: `Season {season}`
5. **Download client** (`Settings > Download Clients > Add > qBittorrent`):

   | Field    | Value              |
   |----------|--------------------|
   | Host     | `qbittorrent`      |
   | Port     | `8081`             |
   | Username | `admin`            |
   | Password | the one from 2.1   |
   | Category | `tv-sonarr`        |

   Leave `Completed Download Handling` enabled (same page, further down).
6. **Indexers**: they appear on their own with the `(Prowlarr)` suffix a few seconds after step 2.2.5. If nothing shows up, check the API key in Prowlarr.
7. **Notifications** (`Settings > Connect`): Telegram is configured. It needs a bot token and a chat id, see the [Sonarr docs](https://wiki.servarr.com/sonarr/settings#connections).

### 2.4 Radarr

URL: `http://localhost:7878`

Same as Sonarr, with these values:

- Root folder: `/data/complete/movies`
- Naming: `{Movie Title} ({Release Year}) {Quality Full}`, folder `{Movie Title} ({Release Year})`
- qBittorrent client: category `radarr`
- Hardlinks: same option, to be ticked

### 2.5 Plex

URL: `http://localhost:32400/web`

Plex runs with `network_mode: host`, there is no port mapping to manage.

1. **Claim.** On the very first start only, Plex attaches to your account through `PLEX_CLAIM` in `.env`. The token comes from [plex.tv/claim](https://www.plex.tv/claim) and **expires after 4 minutes**: grab it, paste it in `.env`, run `docker compose up -d plex` immediately. Once the server is claimed, the variable is ignored.
2. **Libraries** (`Settings > Manage > Libraries`):

   | Library   | Type     | Folder                   |
   |-----------|----------|--------------------------|
   | Movies    | Movies   | `/media/complete/movies` |
   | TV Shows  | TV Shows | `/media/complete/tv`     |

3. **Remote access** (`Settings > Remote Access`): optional. If you go through WireGuard you are virtually on the LAN, Plex remote access is not needed.
4. **Token for Homepage**: see [3.3](#33-get-the-plex-token).

### 2.6 Homepage

URL: `http://localhost`

Homepage is configured through YAML files in `${PATH_CONF}/config/homepage/`. They are reloaded live, no restart needed.

| File             | Role                                                          |
|------------------|---------------------------------------------------------------|
| `settings.yaml`  | title, language, group layout                                 |
| `services.yaml`  | service tiles, with their widgets                             |
| `widgets.yaml`   | top bar: resources, search, clock, weather                    |
| `bookmarks.yaml` | plain links                                                   |
| `docker.yaml`    | connection to the Docker socket for container status          |

**Allowed hosts.** Homepage rejects any request whose `Host` header is not in `HOMEPAGE_ALLOWED_HOSTS` (in `.env`). Put the LAN IP with the port and the hostname, comma separated.

**Docker integration.** For tiles to show container status, `docker.yaml` must declare the socket, and each service must reference that name:

```yaml
# docker.yaml
zerepflix:
  socket: /var/run/docker.sock
```

```yaml
# services.yaml, on each tile
server: zerepflix
container: plex
```

A `server: localhost` without a `localhost` entry in `docker.yaml` shows nothing.

**Secrets.** Widgets need the Sonarr, Radarr and Prowlarr API keys, the Plex token and the qBittorrent credentials. Instead of writing them in `services.yaml`, Homepage replaces `{{HOMEPAGE_VAR_XXX}}` with the container's `HOMEPAGE_VAR_XXX` environment variable. Pass those variables in the compose file from `.env`, and the YAML files can be versioned.

Example `services.yaml` matching the stack:

```yaml
- Seedbox:
    - Sonarr:
        icon: sonarr.png
        href: http://localhost:8989
        server: zerepflix
        container: sonarr
        widget:
            type: sonarr
            url: http://sonarr:8989
            key: "{{HOMEPAGE_VAR_SONARR_KEY}}"
    - Radarr:
        icon: radarr.png
        href: http://localhost:7878
        server: zerepflix
        container: radarr
        widget:
            type: radarr
            url: http://radarr:7878
            key: "{{HOMEPAGE_VAR_RADARR_KEY}}"
    - Prowlarr:
        icon: prowlarr.png
        href: http://localhost:9696
        server: zerepflix
        container: prowlarr
        widget:
            type: prowlarr
            url: http://prowlarr:9696
            key: "{{HOMEPAGE_VAR_PROWLARR_KEY}}"

- Movie Management:
    - Plex:
        icon: plex.png
        href: https://app.plex.tv
        server: zerepflix
        container: plex
        widget:
            type: plex
            url: http://<SERVER_IP>:32400
            key: "{{HOMEPAGE_VAR_PLEX_TOKEN}}"
    - qBittorrent:
        icon: qbittorrent.png
        href: http://localhost:8081
        server: zerepflix
        container: qbittorrent
        widget:
            type: qbittorrent
            url: http://qbittorrent:8081
            username: admin
            password: "{{HOMEPAGE_VAR_QBT_PASSWORD}}"
            enableLeechProgress: true
    - JOAL:
        icon: joal.png
        href: http://localhost:3231/<JOAL_UI_PREFIX>/ui/#/
```

Notes:

- Widget `url:` entries use container names, Homepage queries the services from the Docker network. `href:` entries stay on `localhost`, your browser opens them. Plex is the exception: with `network_mode: host` it has no name on the Docker network, and `localhost` from inside the Homepage container is Homepage itself. Its widget must therefore target the server's LAN IP.
- The JOAL link depends on `JOAL_UI_PREFIX` defined in `joal/.env`. If you change the prefix, change the link.
- The `resources` widget with `disk: /mnt/Zerepflix` in `widgets.yaml` requires the `${PATH_MEDIA}:/mnt/Zerepflix` mount from the compose file. Without the widget, the mount is useless.
- The `openweathermap` and `weatherapi` providers in `settings.yaml` are only used if a weather widget relies on them. The `openmeteo` widget needs no key.

### 2.7 WireGuard

WireGuard is a **VPN server** to reach the stack from outside, as if you were on the LAN. It does not route torrent traffic through a VPN.

Variables in `.env`:

| Variable              | Role                                                                   |
|-----------------------|------------------------------------------------------------------------|
| `WIREGUARD_SERVERURL` | public hostname written into the peer configs. Use a DDNS name, a public IP changes. |
| `WIREGUARD_PEERS`     | list of clients, comma separated. One folder per peer.                 |

Network: the server listens on `51820/udp`, peers get an address in `10.13.13.0/24`, the tunnel DNS is `10.13.13.1` and all traffic goes through the tunnel (`AllowedIPs 0.0.0.0/0`).

On the router: **forward port 51820/udp** to the server.

Generated configs live in `${PATH_CONF}/config/wireguard/peer_<name>/`: `peer_<name>.conf` to import in the client, `peer_<name>.png` to scan from the mobile app. To show a QR code in the terminal:

```shell
docker exec wireguard /app/show-peer iphone
```

**Warning**: changing `WIREGUARD_SERVERURL` or `WIREGUARD_PEERS` regenerates every peer config when the container restarts. You then have to re-import the configs on every device.

### 2.8 JOAL

Separate stack in `joal/`, see the README. Configuration points in `${PATH_JOAL}/conf/`:

- `config.json`: simulated upload speed (`minUploadRate`, `maxUploadRate` in kB/s), number of torrents seeded in parallel, emulated client.
- `clients/`: definition files of the emulated BitTorrent client. The one referenced in `config.json` must exist.
- `torrents/`: drop the `.torrent` files whose ratio you want to raise here.

The UI is protected by `JOAL_UI_PREFIX` and `JOAL_UI_TOKEN` from `joal/.env`: `http://localhost:3231/<prefix>/ui?token=<token>`.

---

## 3. Recurring procedures

### 3.1 Find an API key (Sonarr, Radarr, Prowlarr)

In the UI: `Settings > General > Security > API Key`. Or without the UI:

```shell
grep ApiKey ${PATH_CONF}/config/sonarr/config.xml
```

### 3.2 Reset a Sonarr, Radarr or Prowlarr password

Edit `${PATH_CONF}/config/<service>/config.xml`, replace `<AuthenticationMethod>Forms</AuthenticationMethod>` with `None`, restart the container. You get in without a login, then set `Forms` authentication again in `Settings > General`.

### 3.3 Get the Plex token

In Plex Web, open any media item, `...` menu > `Get Info` > `View XML`. The token is the `X-Plex-Token` parameter in the URL. Details on [plexopedia](https://www.plexopedia.com/plex-media-server/general/plex-token/).

### 3.4 Reset the qBittorrent password

```shell
docker compose stop qbittorrent
sed -i '/WebUI\\Password_PBKDF2/d' ${PATH_CONF}/config/qbittorrent/qBittorrent.conf
docker compose start qbittorrent
docker logs qbittorrent 2>&1 | grep -i password
```

Remember to update the password in Sonarr, Radarr and Homepage.

### 3.5 Add a WireGuard peer

Add the name to `WIREGUARD_PEERS`, then `docker compose up -d wireguard`. Existing peers are regenerated too, see the warning in 2.7.

### 3.6 Check that hardlinks work

The link count (second column) must be `2` for a file present in both `downloads/` and `complete/`:

```shell
ls -l "${PATH_MEDIA}/complete/tv/Some Show/Season 1/"
```

If it is `1`, the file was copied: check the hardlink option in Sonarr/Radarr and that `downloads/` and `complete/` are on the same disk.

### 3.7 Back up

Everything that matters is in `${PATH_CONF}/config`. Stop the stack first to get consistent SQLite databases:

```shell
docker compose stop
tar czf zerepflix-config-$(date +%F).tgz -C "${PATH_CONF}" config
docker compose start
```

Sonarr, Radarr and Prowlarr also keep their own backups in `<service>/Backups/`, restorable from `System > Backup`.