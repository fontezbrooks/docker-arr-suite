# Fork Readiness Plan

## Context

You forked an automated media-management Docker Compose stack and want to understand what it does before adapting it for your own environment. The repo currently defines a Traefik + Gluetun + qBittorrent + *arr stack, with optional Plex and Jellyfin compose files.

Target direction from user:
- Deployment target is a single server: Windows Server preferred unless Docker/Compose compatibility issues make Linux necessary.
- Storage should move away from NFS toward local paths/bind mounts, rooted at `/srv/media-stack`.
- Jellyfin is the desired media server; Plex should remain disabled/optional.
- User owns a domain managed in Cloudflare.
- Only Jellyseerr should be publicly reachable for adding/requesting media remotely.
- Jellyfin streaming and admin tools such as Sonarr, Radarr, Prowlarr, and qBittorrent should not be publicly exposed by default.
- No VPN provider has been chosen yet; torrent traffic still needs a VPN/Gluetun decision before real use.
- The fork is private-use infrastructure, but it will live on GitHub for repeatability; secrets must not be committed.

Initial findings:
- Main entrypoint is `docker-compose.yml`, which includes `bittorrent-compose.yml` and `arr-compose.yml` by default.
- `plex-compose.yml` and `jellyfin-compose.yml` are present but commented out in the main compose file.
- Original storage was NFS-based throughout, using `${NFS_SERVER}` and `${NFS_VOLUME}` for Docker app data and media paths; this has been converted to local bind mounts.
- Public access is via Traefik hostnames like `sonarr.${DOMAIN}`, with Cloudflare DNS challenge secrets; this needs to be narrowed so only `jellyseerr.${DOMAIN}` is public by default.
- VPN/torrent traffic appears routed through Gluetun; qBittorrent, Sonarr, Radarr, and Prowlarr use `network_mode: service:gluetun`.
- The repo includes real `env/` and `secrets/` file paths. Secret file contents have not been inspected and should be handled carefully.
- `README.md` mentions a `media.sh` script, but no shell script was found in the repo.
- No `.gitignore` exists; `node_modules/`, `.pi/`, package files, and this plan are currently untracked.
- Initial Compose validation warned that `EMAIL_ADDRESS`, `DOMAIN`, `NFS_SERVER`, and `NFS_VOLUME` were unset. After moving to local paths, `NFS_SERVER`/`NFS_VOLUME` are no longer required.
- The files under `env/*.env` originally used YAML-style `KEY: value`; they have been converted to Docker-compatible `KEY=value` syntax and copied to `.example` templates.
- `env/bittorrent.env` originally referenced `/run/secrets/bittorrent-passowrd` while the declared secret is `bittorrent-password`; the typo has been fixed/removed.

## Approach

First, audit the repository as infrastructure configuration rather than application code. Then adapt the fork safely for a single-server Jellyfin setup on Windows Server when feasible: replace NFS volume definitions with local bind mounts under `/srv/media-stack` inside the Linux-container environment, document required variables/secrets, identify broken or risky defaults, enable the Jellyfin/Jellyseerr path, expose only Jellyseerr publicly through Traefik/Cloudflare, keep Jellyfin/admin apps private by default, add missing onboarding docs/examples, and validate the compose configuration before any deployment.

NFS note for documentation: NFS means Network File System, where Docker volumes are stored on another machine such as a NAS. It is useful for centralized/shared storage across multiple hosts, but adds network dependency, permissions complexity, and operational fragility. For a single server, local bind mounts are simpler and usually more reliable.

## Files to modify

Likely documentation/configuration updates after planning:
- `README.md` — update setup instructions for your fork, explain Windows/Linux-container local-path setup, and remove stale `media.sh` reference if appropriate.
- `docker-compose.yml` — include Jellyfin compose, keep Plex disabled, switch shared volumes away from NFS, and keep Traefik listening publicly for Jellyseerr.
- `bittorrent-compose.yml` — switch volumes to local paths, fix qBittorrent/port-forwarder configuration issues, and remove/disable public Traefik exposure for qBittorrent.
- `arr-compose.yml` — switch volumes to local paths, tune *arr services and unpackerr wiring, and remove/disable public Traefik exposure for Sonarr/Radarr/Prowlarr.
- `jellyfin-compose.yml` — enable Jellyfin/Jellyseerr, switch volumes to local paths, expose Jellyseerr publicly, and keep Jellyfin private/local by default.
- `plex-compose.yml` — leave disabled/optional; possibly document as not used.
- `env/*.env` — convert to valid `KEY=value` format and/or create safe templates.
- `secrets/` — establish safe placeholder/example handling and avoid committing real secrets.
- `.gitignore` — add protection for local secrets/env overrides and generated dependencies/artifacts.

## Reuse

Existing pieces to preserve/reuse where possible:
- Compose include structure in `docker-compose.yml` for modular stack selection.
- Existing Traefik and Cloudflare DNS challenge setup in `docker-compose.yml`, but with service labels restricted to Jellyseerr by default.
- Existing Gluetun VPN gateway pattern used by torrent and *arr services.
- Existing Jellyfin/Jellyseerr service definitions in `jellyfin-compose.yml`, especially Jellyseerr as the public request interface.
- LinuxServer.io images for Sonarr/Radarr/qBittorrent and existing healthcheck patterns.
- Existing NFS volume path layout as a guide for the equivalent local directory layout, even though NFS itself should be removed for this setup.

## Open questions

- Windows Server is preferred. Need to confirm the practical Docker runtime path: Docker Desktop/WSL2 where available, or a Linux VM/WSL-backed Docker engine if native Windows containers are not suitable. This stack uses Linux container images, so it should run on Windows only through a Linux-container Docker backend, not native Windows containers.
- Which VPN provider should be used? The current Gluetun defaults are for Private Internet Access, but no provider has been chosen yet.
- Should Jellyfin be accessible only on the LAN, or through a private remote-access tool later such as Tailscale/Cloudflare Tunnel? Public Jellyfin streaming is not a priority now.

## Steps

- [x] Default the plan to Windows Server, but document that the stack uses Linux containers and therefore needs Docker running in Linux-container mode, WSL2, or a Linux VM rather than native Windows containers.
- [x] Inventory all required environment variables and secret files without exposing secret values.
- [x] Replace NFS-backed named volumes with local bind mounts rooted at `/srv/media-stack`.
- [x] Use a local directory layout like `/srv/media-stack/config/<app>`, `/srv/media-stack/data/media`, and `/srv/media-stack/data/torrents`.
- [x] Enable `jellyfin-compose.yml` in `docker-compose.yml`; keep `plex-compose.yml` disabled.
- [x] Keep only Jellyseerr publicly exposed via Traefik/Cloudflare (`jellyseerr.${DOMAIN}`).
- [x] Remove or disable public Traefik labels/routes for qBittorrent, Sonarr, Radarr, Prowlarr, and Jellyfin by default.
- [x] Keep admin UIs accessible locally/LAN-side, or document a later private-access method.
- [x] Convert `env/*.env` files to valid `KEY=value` syntax or create template files and local ignored copies.
- [x] Fix known config typo: `bittorrent-passowrd` -> `bittorrent-password`.
- [x] Decide VPN provider/Gluetun settings before downloading torrents; document provider selection as a prerequisite.
- [x] Validate Docker Compose syntax and interpolation requirements.
- [x] Identify additional configuration bugs, stale docs, security risks, and provider-specific assumptions.
- [x] Add/update documentation, `.gitignore`, and safe examples/templates.
- [x] Validate final compose output before deployment.

## Verification

Planned verification after changes:
- Run Docker Compose config validation with required variables supplied via a safe example/local env file.
- Check that required secret files exist but are not committed with real values.
- Confirm enabled services, networks, and volumes match the chosen deployment model.
- Confirm `NFS_SERVER` and `NFS_VOLUME` are no longer required after local-path conversion.
- Confirm only Jellyseerr has a public Traefik route by default.
- Bring up the stack incrementally: Traefik first, Gluetun after VPN provider configuration, then qBittorrent, then *arr services, then Jellyfin/Jellyseerr.
- Manually verify service health endpoints and Traefik routes.
