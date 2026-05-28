# Personal Media Stack

Docker Compose stack for a private media server with remote media requests.

This fork is configured for:

- **Jellyfin for Windows** as the media server, installed with the official standalone `.exe` installer.
- **Jellyseerr** in Docker as the only public web app, at `https://jellyseerr.${DOMAIN}`.
- **qBittorrent + Sonarr + Radarr + Prowlarr + Unpackerr** in Docker for media automation.
- **Gluetun** as the VPN gateway for torrent/*arr traffic.
- **Traefik + Cloudflare DNS challenge** for public HTTPS.
- A Windows shared storage root such as `D:\MediaStack` / `D:/MediaStack` instead of NFS.

## Important platform note

Windows Server is the preferred target for this fork. Jellyfin itself should be installed directly on Windows with the standalone installer, while the rest of the stack still uses **Linux container images**. The Docker services will not run as native Windows containers.

Use Docker with a Linux-container backend / WSL2, or a small Linux VM on the Windows Server host. Configure Docker bind mounts to use the same Windows media root that Jellyfin uses directly.

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

This fork uses local bind-mounted storage rooted at a Windows folder such as `D:\MediaStack`.

## Directory layout

Create this shared directory layout before first start:

```powershell
New-Item -ItemType Directory -Force `
  D:\MediaStack\config\traefik\letsencrypt, `
  D:\MediaStack\config\gluetun, `
  D:\MediaStack\config\qbittorrent, `
  D:\MediaStack\config\sonarr, `
  D:\MediaStack\config\radarr, `
  D:\MediaStack\config\prowlarr, `
  D:\MediaStack\config\unpackerr, `
  D:\MediaStack\config\jellyseerr, `
  D:\MediaStack\data\media, `
  D:\MediaStack\data\torrents
```

Set `MEDIA_STACK_ROOT=D:/MediaStack` in `.env`. Docker uses forward slashes for the bind-mount root; Jellyfin for Windows should use the normal Windows path, for example `D:\MediaStack\data\media`.

## Jellyfin for Windows

Install Jellyfin directly on Windows instead of running it in Docker.

Current official Windows standalone installer URL:

<https://repo.jellyfin.org/files/server/windows/latest-stable/amd64/jellyfin_10.11.10_windows-x64.exe>

After installation:

1. Start Jellyfin and complete the first-run wizard.
2. Add media libraries that point at `D:\MediaStack\data\media` or subfolders inside it.
3. Make Jellyfin reachable on the LAN, normally at `http://<windows-host-lan-ip>:8096`.
4. In Jellyfin Dashboard → Networking, keep the HTTP port at `8096` unless you intentionally change it.
5. Allow inbound TCP `8096` through Windows Firewall for your LAN/private network profile.

Jellyfin is not routed publicly through Traefik in this repo. For remote Jellyfin access later, prefer a private-access method such as Tailscale, WireGuard, or another VPN rather than exposing Jellyfin publicly by default.

## Jellyseerr setup with external Jellyfin

Jellyseerr remains a Docker service. It does not depend on a Jellyfin container; configure the Windows-host Jellyfin server in the Jellyseerr web UI after first startup.

Recommended Jellyseerr connection value:

```text
http://<windows-host-lan-ip>:8096
```

Do not use `localhost` from inside the Jellyseerr container, because that refers to the container itself, not the Windows host. If the host LAN IP changes, update the Jellyfin server URL in Jellyseerr settings.

If Jellyseerr cannot connect, first confirm Jellyfin is healthy from another LAN device at `http://<windows-host-lan-ip>:8096`, then check Windows Firewall/private-network rules for inbound TCP `8096`.

## Public/private access model

Public through Traefik/Cloudflare:

- `jellyseerr.${DOMAIN}` only

Not publicly exposed by default:

- Jellyfin for Windows (`http://<windows-host-lan-ip>:8096`, LAN only)
- qBittorrent
- Sonarr
- Radarr
- Prowlarr

Local-only Docker bindings:

- qBittorrent: `http://127.0.0.1:8088`
- Sonarr: `http://127.0.0.1:8989`
- Radarr: `http://127.0.0.1:7878`
- Prowlarr: `http://127.0.0.1:9696`

For safer remote admin access later, prefer Tailscale, WireGuard, SSH tunneling, or another private-access method instead of public Traefik routes.

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
| `MEDIA_STACK_ROOT` | Shared Windows stack root for Docker bind mounts; default example `D:/MediaStack` |

## Environment files

The active local files are `env/*.env`. They are ignored by git. Safe templates are committed as `env/*.env.example`.

To initialize local env files from templates:

```bash
for f in env/*.env.example; do cp "$f" "${f%.example}"; done
```

All env files use Docker-compatible `KEY=value` syntax. `env/jellyseerr.env` is Jellyseerr-only; Jellyfin is configured in the Windows app and in the Jellyseerr web UI, not via Docker environment variables.

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
- `jellyseerr-compose.yml` included.
- No Docker `jellyfin` service.
- `plex-compose.yml` disabled.
- Only Jellyseerr has public Traefik router labels.

## Startup order

Recommended first deployment sequence:

1. Create `D:\MediaStack` directories and configure `.env` with `MEDIA_STACK_ROOT=D:/MediaStack`.
2. Install and configure Jellyfin for Windows; confirm `http://<windows-host-lan-ip>:8096` works from the LAN.
3. Validate Docker Compose syntax.
4. Start Traefik and confirm certificates can be issued.
5. Configure VPN provider and start Gluetun.
6. Start qBittorrent.
7. Start Sonarr/Radarr/Prowlarr.
8. Start Jellyseerr and connect it to `http://<windows-host-lan-ip>:8096` in the web UI.

Example full Docker start after configuration:

```bash
docker compose up -d
```

## Known follow-up tasks

- Choose the VPN provider and finalize Gluetun provider-specific settings.
- Confirm qBittorrent Web UI credentials and keep `env/bittorrent.env` in sync.
- Add Radarr/Sonarr API keys to local secret files after initial app setup.
- Keep the Windows host LAN IP stable with a DHCP reservation or static IP so Jellyseerr can reliably reach Jellyfin.
