# media-stack

A self-hosted **torrent automation stack** for Docker Compose: a **BitTorrent
client routed through a VPN kill switch**, with automatic search, download,
import and subtitle fetching. One `docker compose up -d`, no secrets in the
repository, identical on macOS and Linux — laptop, homelab server, NAS or VPS.

## What's in the stack

| Service | Role |
| --- | --- |
| [gluetun](https://github.com/qdm12/gluetun) | VPN client (OpenVPN or WireGuard) acting as the network gateway and kill switch |
| [Transmission](https://transmissionbt.com/) | BitTorrent client, running entirely inside the VPN tunnel |
| [Prowlarr](https://prowlarr.com/) | Indexer/tracker manager, syncs indexers to the apps below |
| [Sonarr](https://sonarr.tv/) | TV series: monitors, searches and imports episodes |
| [Radarr](https://radarr.video/) | Movies: monitors, searches and imports releases |
| [Bazarr](https://www.bazarr.media/) | Subtitles for whatever Sonarr and Radarr import |

🇫🇷 [Version française](README.fr.md)

## What makes this setup different

- **Transmission has no route to the Internet except the VPN.** It shares
  gluetun's network namespace (`network_mode: service:gluetun`), so if the
  tunnel drops, Transmission goes offline. The kill switch is structural: it
  holds even when nothing is configured to enforce it.
- **Everything is bound to `127.0.0.1` by default.** No web UI is exposed to
  your LAN or the Internet until you deliberately change `BIND_ADDRESS`.
- **One shared mount, so hardlinks work.** Downloads and the media library live
  under the same host directory mounted at `/data`, which lets Sonarr and
  Radarr import by hardlink: no copy, no doubled disk usage, seeding continues.
- **No credentials in the repo.** `config/` and `.env` are git-ignored; the
  repository ships only `.env.example`.

## Requirements

- Docker Engine (or Docker Desktop) with Compose v2.24+
- A VPN subscription supported by
  [gluetun](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers)
  — the defaults here target CyberGhost/OpenVPN, but any provider works
- Enough disk space under the directory you point `DOWNLOADS_DIR` at

## Install

```bash
git clone https://github.com/maax6/media-stack.git
cd media-stack
cp .env.example .env
```

Edit `.env`:

| Variable | Notes |
| --- | --- |
| `PUID` / `PGID` | `id -u` / `id -g`. Linux: usually `1000/1000`. macOS: usually `501/20`. |
| `TZ` | e.g. `Europe/Paris` |
| `DOWNLOADS_DIR` | The single host directory every service shares |
| `VPN_SERVICE_PROVIDER`, `OPENVPN_USER`, `OPENVPN_PASSWORD` | From your VPN dashboard |
| `TRANSMISSION_USER` / `TRANSMISSION_PASSWORD` | Required — do not leave the defaults |

Create the media directories and, **for providers that require a client
certificate (CyberGhost among them)**, drop the certificate pair in place —
gluetun will not start without it:

```bash
mkdir -p "$DOWNLOADS_DIR"/{torrents,media/movies,media/tv}
mkdir -p config/gluetun
cp /path/to/client.crt /path/to/client.key config/gluetun/
```

Start it:

```bash
docker compose up -d
docker compose ps
```

Confirm the tunnel is actually up and that Transmission is behind it — this
must return the VPN's IP, not yours:

```bash
docker compose logs gluetun | grep -i "public ip"
```

## Web UIs

All bound to `127.0.0.1` by default; ports are configurable in `.env`.

| Application | URL |
| --- | --- |
| Transmission | http://127.0.0.1:19091 |
| Sonarr | http://127.0.0.1:18989 |
| Radarr | http://127.0.0.1:17878 |
| Prowlarr | http://127.0.0.1:19696 |
| Bazarr | http://127.0.0.1:16767 |

## First-run configuration

1. **Transmission** — set the download directory to `/data/torrents`.
2. **Sonarr** — root folder `/data/media/tv`. Add the Transmission download
   client with host `transmission`, port `9091`. Leave *Remote Path Mapping*
   empty: every container sees the identical `/data` path. Enable hardlinks in
   *Media Management*.
3. **Radarr** — same, with root folder `/data/media/movies`.
4. **Prowlarr** — add your indexers, then under *Settings → Apps* add Sonarr
   (`http://sonarr:8989`) and Radarr (`http://radarr:7878`) so indexers sync
   automatically.
5. **Bazarr** — connect it to Sonarr (`http://sonarr:8989`) and Radarr
   (`http://radarr:7878`), then add subtitle providers.

## Security model — read this before exposing anything

**What is protected:** all peer-to-peer traffic. Transmission cannot reach the
Internet outside the tunnel, by design.

**What is not:**

- **Sonarr, Radarr, Prowlarr and Bazarr use your real IP.** They talk to
  metadata services and indexer APIs directly. Torrent traffic — the part that
  matters for peer visibility — stays inside the VPN. Routing Prowlarr through
  gluetun too is possible but deliberately not done here: shared VPN exit IPs
  get captcha'd and blocked by many indexers, Docker service names stop
  resolving inside gluetun's network namespace, and the gain is narrow since no
  peer traffic ever leaves that container.
- **These apps have no meaningful authentication hardening.** Keep
  `BIND_ADDRESS=127.0.0.1`. If you need remote access, use a VPN back into your
  network (Tailscale, WireGuard) or a reverse proxy that terminates TLS and
  enforces authentication. Do not port-forward these to the Internet.
- **`config/` holds live secrets** once you have run the stack: API keys, the
  Transmission password hash, and your VPN private key. It is git-ignored — if
  you back it up, treat that backup as sensitive.

**No incoming peer port.** Transmission's peer port is deliberately not
published, so the client is outbound-only. Ratios on private trackers will
suffer. If your VPN provider supports port forwarding, gluetun can be
configured for it — see the gluetun wiki.

## Everyday commands

```bash
docker compose logs -f            # follow all logs
docker compose logs -f gluetun    # tunnel state, public IP, reconnects
docker compose pull && docker compose up -d   # update images
docker compose down               # stop; config and media are untouched
```

## Moving to another machine

Copy `.env` and `config/` over by a secure channel (both contain secrets), then
`git clone` and `docker compose up -d`. Adjust `PUID`/`PGID` and
`DOWNLOADS_DIR` for the new host. Alternatively, start fresh: the stack builds
its own `config/` on first run, and you reconfigure the apps by hand.

## FAQ

### Does the kill switch actually hold when the VPN drops?

Yes, because it is not a feature that has to run. Transmission has no network
interface of its own: it shares gluetun's network namespace. When the tunnel
goes down, the only route out disappears with it. Verify with
`docker compose logs gluetun | grep -i "public ip"`.

### Which VPN providers work?

Every provider gluetun supports — around 60, including NordVPN, Mullvad,
ProtonVPN, Surfshark, Private Internet Access, CyberGhost and any custom
OpenVPN or WireGuard configuration. Set `VPN_SERVICE_PROVIDER` in `.env`; see
the [gluetun provider list](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers).

### Can I use WireGuard?

Yes. Set `VPN_TYPE=wireguard` and fill `WIREGUARD_PRIVATE_KEY` and
`WIREGUARD_ADDRESSES` in `.env`. It is faster than OpenVPN on most hardware.

### Can I swap Transmission for qBittorrent or Deluge?

Yes. Replace the `transmission` service image with
`lscr.io/linuxserver/qbittorrent` or `lscr.io/linuxserver/deluge`, keep
`network_mode: "service:gluetun"`, and update the published port and
`FIREWALL_INPUT_PORTS` to match the new web UI port.

### Why are my imports copying files instead of hardlinking?

Hardlinks only work inside a single filesystem. Downloads and the media library
must both live under `DOWNLOADS_DIR`, and every container must see them at the
same `/data` path — which is why this stack uses one shared mount and no Remote
Path Mapping. Also enable hardlinks in *Media Management* in Sonarr and Radarr.

### Does it run on a Raspberry Pi, a Synology NAS or a VPS?

Yes. All images are multi-arch (amd64 and arm64). On Linux set `PUID`/`PGID` to
`id -u` / `id -g`, usually `1000/1000`. `/dev/net/tun` must be available, which
rules out some restricted container hosts.

### How do I reach the web UIs from another machine?

Keep `BIND_ADDRESS=127.0.0.1` and connect back into your network with
Tailscale or WireGuard. Exposing these apps directly is unsafe: they were never
designed to face the Internet.

### Do I need port forwarding?

Only to accept incoming peers. This stack keeps the peer port unpublished, so
Transmission is outbound-only and upload ratios stay low. If your provider
supports port forwarding, gluetun can request one — see its wiki.

## Intended use

BitTorrent is a distribution protocol, and plenty of what it distributes is
free to share: Linux and BSD images (Debian, Ubuntu, Arch, Fedora), the
[Internet Archive](https://archive.org/) collections, public-domain films,
Creative Commons music and video, open datasets, game and demoscene archives,
and Blender Foundation open movies. This stack automates exactly that kind of
library.

## Legal

This stack is plumbing. It ships with no indexers, no trackers, and no content.
Downloading copyrighted material without authorisation is illegal in most
jurisdictions — what you use it for is on you.

## License

[MIT](LICENSE)
