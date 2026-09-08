# Zerepflix Project

## Quick reference

- **Maintained by**: [Zerep](https://github.com/Zerep34).

## What is Zerepflix?

Zerepflix is a docker-compose stack which runs a complete download Toolbox, for movies and series

## Usage

Follow the following steps to run Zerepflix

- Clone the repository on your server : [https://github.com/Zerep34/Zerepflix](https://github.com/Zerep34/Zerepflix)
- In the folder rename the file .env.tpl in .env and then replace the variables (paths, Plex claim token from [plex.tv/claim](https://www.plex.tv/claim), Homepage allowed hosts, WireGuard peers)
- Then run the following command :
    ```shell
    docker compose up -d
    ```

## UI Access

| Application | URL                 |                                                         Documentation |
|-------------|---------------------|----------------------------------------------------------------------:|
| Homepage    | localhost:80        |                            [Docs Homepage](https://gethomepage.dev/) |
| Radarr      | localhost:7878      |                     [Wiki Radarr](https://wiki.servarr.com/en/radarr) |
| Sonarr      | localhost:8989      |                        [Wiki Sonarr](https://wiki.servarr.com/sonarr) |
| Prowlarr    | localhost:9696      |                    [Wiki Prowlarr](https://wiki.servarr.com/prowlarr) |
| qBittorrent | localhost:8081      |             [Wiki qBittorrent](https://github.com/qbittorrent/qBittorrent/wiki) |
| Plex        | localhost:32400/web |                    [Support Plex](https://support.plex.tv/articles/) |
| WireGuard   | udp 51820 (no UI)   | [Docs WireGuard](https://docs.linuxserver.io/images/docker-wireguard) |

### qBittorrent first login

qBittorrent generates a temporary password on first start. Get it with:

```shell
docker logs qbittorrent 2>&1 | grep -i password
```

Then change it in the Web UI (Tools > Options > Web UI).

### WireGuard

WireGuard is a VPN server to reach the stack remotely from your devices (peers in `WIREGUARD_PEERS`).
The server's public IP is detected at startup and written into the peer configs. Forward port 51820/udp on your router.
Peer configs and QR codes are generated in `${PATH_CONF}/config/wireguard/peer_<name>/`.

## JOAL (torrent ratio)

[JOAL](https://github.com/anthonyraymond/joal) fakes upload to keep your ratio on private trackers. It runs as a separate stack in the `joal/` folder:

- Rename `joal/.env.tpl` in `joal/.env` and set the path, UI prefix and secret token
- Drop your `.torrent` files in `${PATH_JOAL}/conf/torrents`
- Run `docker compose up -d` from the `joal/` folder
- UI: `http://localhost:3231/<JOAL_UI_PREFIX>/ui?token=<JOAL_UI_TOKEN>`

## Application configuration

See [docs/CONFIGURATION.md](docs/CONFIGURATION.md) for the expected folder layout, the per-service setup in order, and the recurring procedures (API keys, password reset, WireGuard peers, backups).

General background on this kind of stack: [sebgl/htpc-download-box](https://github.com/sebgl/htpc-download-box)
