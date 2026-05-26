# Personal Media Stack

Docker Compose stack for a private media server with remote media requests.

This fork is configured for:

- **Jellyfin** as the media server.
- **Jellyseerr** as the only public web app, at `https://jellyseerr.${DOMAIN}`.
- **qBittorrent + Sonarr + Radarr + Prowlarr + Unpackerr** for media automation.
- **Gluetun** as the VPN gateway for torrent/*arr traffic.
- **Traefik + Cloudflare DNS challenge** for public HTTPS.
- Local storage under `/srv/media-stack` instead of NFS.

## Important platform note

Windows Server is the preferred target for this fork, but this stack uses **Linux container images**. It will not run as native Windows containers.

Use one of these approaches:

- Docker with a Linux-container backend / WSL2, where available.
- A small Linux VM on the Windows Server host.
- A plain Linux server, if Windows container support becomes a blocker.

Inside the Linux-container environment, the stack expects paths like `/srv/media-stack`.

## What is NFS, and why this fork does not use it?

The upstream compose files used NFS-backed Docker volumes. NFS means **Network File System**: Docker stores config/media on another machine, commonly a NAS.

Pros:

- Useful if media already lives on a NAS.
- Can share one storage backend across multiple servers.
- Centralized storage and backups.

Cons:

- More moving parts.
- Depends on network reliability.
- Permissions can be painful.
- Usually unnecessary for a single-server setup.

This fork uses local bind-mounted storage rooted at `/srv/media-stack`.

## Directory layout

Create these directories before first start:

```bash
sudo mkdir -p \
  /srv/media-stack/config/{traefik/letsencrypt,gluetun,qbittorrent,sonarr,radarr,prowlarr,unpackerr,jellyfin,jellyfin-cache,jellyseerr} \
  /srv/media-stack/data/{media,torrents}

sudo chown -R 1000:1000 /srv/media-stack
```

If your Docker Linux backend runs as a different user, adjust `PUID`, `PGID`, and ownership accordingly.

## Public/private access model

Public through Traefik/Cloudflare:

- `jellyseerr.${DOMAIN}` only

Not publicly exposed by default:

- Jellyfin
- qBittorrent
- Sonarr
- Radarr
- Prowlarr

Local-only bindings:

- qBittorrent: `http://127.0.0.1:8088`
- Sonarr: `http://127.0.0.1:8989`
- Radarr: `http://127.0.0.1:7878`
- Prowlarr: `http://127.0.0.1:9696`
- Jellyfin: `http://127.0.0.1:8096`

For LAN access, change the relevant port binding from `127.0.0.1:PORT:PORT` to `PORT:PORT`. For safer remote admin access later, prefer Tailscale, WireGuard, SSH tunneling, or another private-access method instead of public Traefik routes.

## Required top-level variables

Copy `.env.example` to `.env` and customize:

```bash
cp .env.example .env
```

Required values:

| Variable | Purpose |
| --- | --- |
| `DOMAIN` | Your Cloudflare-managed domain, e.g. `example.com` |
| `EMAIL_ADDRESS` | Email used by Let's Encrypt |
| `MEDIA_STACK_ROOT` | Local stack root; default `/srv/media-stack` |

## Environment files

The active local files are `env/*.env`. They are ignored by git. Safe templates are committed as `env/*.env.example`.

To initialize local env files from templates:

```bash
for f in env/*.env.example; do cp "$f" "${f%.example}"; done
```

All env files use Docker-compatible `KEY=value` syntax.

## Required secret files

Secret files live in `secrets/*.secret` and are ignored by git. Do not commit real values.

Create these files locally:

| File | Purpose |
| --- | --- |
| `secrets/cloudflare-token.secret` | Cloudflare DNS API token for Traefik certificates |
| `secrets/cloudflare-email.secret` | Cloudflare account email |
| `secrets/openvpn_user.secret` | VPN/OpenVPN username, after choosing a provider |
| `secrets/openvpn_password.secret` | VPN/OpenVPN password, after choosing a provider |
| `secrets/bittorrent-user.secret` | qBittorrent Web UI username reference |
| `secrets/bittorrent-password.secret` | qBittorrent Web UI password reference |
| `secrets/radarr-api.secret` | Radarr API key for Unpackerr |
| `secrets/sonarr-api.secret` | Sonarr API key for Unpackerr |

Templates exist as `secrets/*.secret.example`.

## VPN prerequisite

Do **not** download torrents until you choose and configure a VPN provider in `env/gluetun.env`.

The previous config assumed Private Internet Access. This fork leaves provider selection explicit. Pick a provider supported by Gluetun and update:

- `VPN_SERVICE_PROVIDER`
- `VPN_TYPE`
- provider-specific server/location variables
- OpenVPN/WireGuard credential secret files
- port-forwarding settings, if your provider supports them

See the Gluetun wiki for provider-specific settings: <https://github.com/qdm12/gluetun-wiki>

## Validate configuration

After creating `.env`, env files, and secret files:

```bash
docker compose config --quiet
```

To inspect the rendered compose without starting containers:

```bash
docker compose config
```

Expected defaults:

- No `NFS_SERVER` or `NFS_VOLUME` interpolation required.
- `jellyfin-compose.yml` included.
- `plex-compose.yml` disabled.
- Only Jellyseerr has public Traefik router labels.

## Startup order

Recommended first deployment sequence:

1. Validate compose syntax.
2. Start Traefik and confirm certificates can be issued.
3. Configure VPN provider and start Gluetun.
4. Start qBittorrent.
5. Start Sonarr/Radarr/Prowlarr.
6. Start Jellyfin and Jellyseerr.

Example full start after configuration:

```bash
docker compose up -d
```

## Known follow-up tasks

- Choose the VPN provider and finalize Gluetun provider-specific settings.
- Confirm qBittorrent Web UI credentials and keep `env/bittorrent.env` in sync.
- Add Radarr/Sonarr API keys to local secret files after initial app setup.
- Decide later whether Jellyfin should be LAN-accessible or private-remote-accessible.
