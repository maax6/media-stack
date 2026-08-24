# media-stack

A small, self-hosted media automation stack: **Transmission** behind a VPN kill
switch, plus **Sonarr**, **Radarr**, **Prowlarr** and **Bazarr**. One
`docker compose up -d`, no secrets in the repository, works the same on macOS
and Linux.

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

## Legal

This stack is plumbing. It ships with no indexers, no trackers, and no content.
Downloading copyrighted material without authorisation is illegal in most
jurisdictions — what you use it for is on you.

## License

[MIT](LICENSE)
