# Windows Jellyfin Executable Plan

## Context

The repository currently runs Jellyfin as a Docker Compose service in `jellyfin-compose.yml`, while the user wants to follow Jellyfin's Windows guidance by installing Jellyfin with the official Windows standalone installer (`jellyfin_10.11.10_windows-x64.exe`) and keeping the rest of the *arr/qBittorrent/Gluetun/Traefik/Jellyseerr stack in Docker.

Initial findings:
- `docker-compose.yml` includes `jellyfin-compose.yml` along with the rest of the Docker stack.
- `jellyfin-compose.yml` currently defines both `jellyfin` and `jellyseerr` Docker services.
- `jellyseerr` currently has `depends_on: jellyfin` and is publicly routed through Traefik at `jellyseerr.${DOMAIN}`.
- Jellyfin's Docker service uses Docker named volumes bound under `${MEDIA_STACK_ROOT:-/srv/media-stack}/config/jellyfin`, `.../config/jellyfin-cache`, and `.../data/media`.
- The rest of the stack already uses Docker bind-backed volumes under `${MEDIA_STACK_ROOT:-/srv/media-stack}`.
- User confirmed the preferred media layout is a Windows path shared with Docker, e.g. `D:\MediaStack\data\media` for Jellyfin and `MEDIA_STACK_ROOT=D:/MediaStack` for Docker bind mounts.
- User wants Jellyfin to be LAN-accessible and prefers README/manual installer instructions only, not an install helper script.
- `env/jellyfin.env.example` currently contains `JELLYFIN_PublishedServerUrl=https://jellyfin.${DOMAIN}`, which is relevant to the Docker Jellyfin container but conflicts with the current goal because Jellyfin is not publicly routed and would no longer be containerized.
- README currently tells users to create Docker-side Jellyfin config/cache directories and lists Jellyfin at `http://127.0.0.1:8096`; this needs to become Windows-executable guidance rather than Docker-container guidance.
- Current rendered Compose service list includes `jellyfin`; after implementation, validation should confirm that service is gone while `jellyseerr` remains.

## Approach

Keep Docker Compose for the rest of the stack, but remove/disable the Jellyfin Docker service and document Jellyfin as a manually installed Windows-host executable. Preserve Jellyseerr as a Docker service, updating its dependency assumptions so it can connect to Jellyfin running on the Windows host/LAN rather than a Compose service. Change the documented storage root from Linux-first `/srv/media-stack` to a Windows-first shared root such as `D:/MediaStack`, so the Windows Jellyfin executable and Docker containers see the same media files.

## Files to modify

Likely files:
- `docker-compose.yml` — keep the Docker stack intact but include a Jellyseerr-only compose file instead of enabling a Jellyfin container.
- `jellyfin-compose.yml` — either reduce this file to Jellyseerr-only or rename it to `jellyseerr-compose.yml` and update the include; preferred plan is rename for clarity.
- `README.md` — update setup instructions for Windows-host Jellyfin executable, Windows shared media path expectations, Docker `MEDIA_STACK_ROOT` guidance, LAN access, and Jellyseerr connection guidance.
- `.env.example` — change `MEDIA_STACK_ROOT` example/default docs to a Windows-style shared root such as `D:/MediaStack`.
- `env/jellyfin.env.example` — remove Docker-Jellyfin-specific settings or replace with a Jellyseerr-focused env template/comment.

## Reuse

Existing pieces to preserve/reuse:
- Existing modular Compose include structure.
- Existing `jellyseerr` Docker service, Traefik labels, healthcheck, and config volume.
- Existing local media directory layout under `${MEDIA_STACK_ROOT}/data/media` for the Docker stack, remapped to a Windows root such as `D:/MediaStack`.
- Existing README sections describing public/private access and directory setup.

## Open questions

- Resolved: Windows Jellyfin should use a Windows-native media directory shared into Docker via `MEDIA_STACK_ROOT`.
- Resolved: Jellyfin should be LAN-accessible on port `8096`.
- Resolved: Do not add an automated install script; document manual installation with the official Windows installer URL.

## Steps

- [x] Inspect current Jellyfin/Jellyseerr env template for existing Jellyfin URL settings.
- [x] Inspect Jellyseerr image/env support for any supported external Jellyfin URL bootstrap settings.
- [x] Rename `jellyfin-compose.yml` to `jellyseerr-compose.yml` or otherwise make it Jellyseerr-only, then update `docker-compose.yml` include accordingly.
- [x] Remove the Jellyfin Docker service, its `jellyfin`/`cache` volumes, and the obsolete Docker-only `JELLYFIN_PublishedServerUrl` setting.
- [x] Remove Jellyseerr's Docker `depends_on` relationship to the Jellyfin container and replace it with documentation/health guidance for an external LAN Jellyfin instance.
- [x] Update `.env.example` and README directory layout to use a Windows shared root (`D:/MediaStack`) for Docker bind mounts and Windows Jellyfin libraries.
- [x] Document Jellyfin Windows installer URL and post-install configuration to listen on the LAN and point libraries at `D:\MediaStack\data\media`.
- [x] Document how Jellyseerr should be configured in its web UI to connect to the Windows-host Jellyfin URL, e.g. `http://<windows-host-lan-ip>:8096`.
- [x] Validate Compose syntax after the Jellyfin service is removed.

## Verification

- Run `docker compose config` with safe example values to ensure Compose remains valid.
- Confirm the `jellyfin` container is no longer part of the Compose service list.
- Confirm `jellyseerr` remains in Compose and keeps its public Traefik route.
- Manual verification: install/start Jellyfin on Windows, browse to `http://<windows-host-lan-ip>:8096` from another LAN device, configure media libraries from `D:\MediaStack\data\media`, start Docker stack, and connect Jellyseerr to the host Jellyfin URL.
