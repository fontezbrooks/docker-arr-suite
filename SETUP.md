# Deployment Runbook

Step-by-step first-time setup for this stack, starting from a machine with nothing installed.

`README.md` is the reference material — what each service is, which variables exist, what is public. This file is the ordered procedure for standing the stack up.

**Order matters. Each phase ends with a verification step. Do not advance past one that fails.**

---

## Phase 0 — Decisions that block everything else

Resolve all three before touching the host.

### 0.1 Which machine runs this?

This repo targets **Windows Server**: Jellyfin installed natively via the standalone `.exe`, with Docker bind mounts rooted at `D:/MediaStack`. Everything below assumes that.

If the real host is Linux or macOS instead, the Jellyfin-on-Windows premise and every `D:/MediaStack` path change. Rework the paths and the Jellyfin section before starting.

### 0.2 VPN provider

Gluetun requires one. Torrent port forwarding additionally requires a provider that supports it — PIA and ProtonVPN are the commonly used pairings with Gluetun, but verify current support in the [Gluetun wiki](https://github.com/qdm12/gluetun-wiki) before subscribing, since provider support changes.

The stack runs fine without port forwarding. The cost is worse seeding performance, not broken functionality.

### 0.3 Domain on Cloudflare

You need a real domain whose nameservers are delegated to Cloudflare. Traefik uses the Cloudflare DNS-01 ACME challenge and cannot issue certificates without it.

---

## Phase 1 — Host preparation

1. Install **Docker Desktop** with the **WSL2 backend** enabled (Settings → General → "Use WSL 2 based engine").

   These are Linux container images. Native Windows containers will not run them.

2. Share the storage drive with Docker: **Settings → Resources → File Sharing**, add `D:`.

   Skipping this causes bind mounts to fail on first start.

3. Create the directory tree. This extends the layout in `README.md` with the library subfolders Sonarr and Radarr need as root folders:

   ```powershell
   New-Item -ItemType Directory -Force `
     D:\MediaStack\config\gluetun, `
     D:\MediaStack\config\qbittorrent, `
     D:\MediaStack\config\sonarr, `
     D:\MediaStack\config\radarr, `
     D:\MediaStack\config\prowlarr, `
     D:\MediaStack\config\unpackerr, `
     D:\MediaStack\config\jellyseerr, `
     D:\MediaStack\data\media\movies, `
     D:\MediaStack\data\media\tv, `
     D:\MediaStack\data\torrents
   ```

   Traefik's Let's Encrypt store is deliberately absent — it is a Docker-managed volume, because Traefik requires `acme.json` at mode `0600`, which Windows-drive bind mounts cannot provide reliably.

**Verify:** `docker run --rm hello-world` succeeds and `docker compose version` reports v2 or newer.

---

## Phase 2 — Cloudflare

1. Delegate the domain's nameservers to Cloudflare. Wait until the zone shows **Active**.

2. Create a **scoped API token**: My Profile → API Tokens → Create Token → *Edit zone DNS*. Grant:
   - `Zone → DNS → Edit`
   - `Zone → Zone → Read`

   Scope it to the single zone. Do not use the Global API Key.

3. Add a DNS record: `jellyseerr` → your public IP. Leave it **proxied (orange cloud)** so your home IP is not exposed. With proxying on, set SSL/TLS mode to **Full (strict)** — Traefik serves a real Let's Encrypt certificate, so strict mode is correct.

4. Forward router ports **80** and **443** (TCP) to the host.

**Verify:** `nslookup jellyseerr.yourdomain.com` resolves.

---

## Phase 3 — Configuration files

From the repository root:

```bash
cp .env.example .env
for f in env/*.env.example; do cp "$f" "${f%.example}"; done
```

Edit `.env`:

| Variable | Value |
| --- | --- |
| `DOMAIN` | your Cloudflare-managed domain |
| `EMAIL_ADDRESS` | address for Let's Encrypt notices |
| `MEDIA_STACK_ROOT` | `D:/MediaStack` — forward slashes |

Edit `env/gluetun.env`: set `VPN_SERVICE_PROVIDER`, `VPN_TYPE`, and any server/country variables your provider requires. Set `VPN_PORT_FORWARDING=off` unless your provider supports forwarding.

### Secret files

Write them **without a trailing newline**. A stray `\n` inside an API token is a common cause of authentication failures that look like a wrong token:

```bash
printf '%s' 'YOUR_CLOUDFLARE_TOKEN' > secrets/cloudflare-token.secret
printf '%s' 'you@example.com'       > secrets/cloudflare-email.secret
printf '%s' 'VPN_USERNAME'          > secrets/openvpn_user.secret
printf '%s' 'VPN_PASSWORD'          > secrets/openvpn_password.secret
printf '%s' 'placeholder'           > secrets/sonarr-api.secret
printf '%s' 'placeholder'           > secrets/radarr-api.secret
```

Sonarr and Radarr API keys do not exist yet. Placeholders now; real values in Phase 6.

**Security check:** `.env`, `env/*.env`, and `secrets/*.secret` are gitignored. Run `git status --short` before any commit and confirm none of them appear.

**Verify:** `docker compose config --quiet` exits 0.

---

## Phase 4 — Traefik, alone

```bash
docker compose up -d traefik
docker logs -f traefik
```

Certificate issuance takes a minute or two: Cloudflare DNS propagation plus the configured 20-second pre-check delay.

**Verify:** no ACME errors in the logs, and:

```bash
docker inspect --format='{{.State.Health.Status}}' traefik
```

returns `healthy`.

If you hit Let's Encrypt rate limits while debugging, temporarily add the staging CA to the Traefik `command:` block:

```yaml
- --certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory
```

Remove it and delete the `letsencrypt` volume before going live, or you will serve an untrusted staging certificate.

---

## Phase 5 — VPN gateway

```bash
docker compose up -d gluetun
docker logs gluetun | grep -i "public ip"
```

**Verify:** the logged public IP is the **VPN's**, not your home IP.

> **Stop here if it matches your home IP.** Every torrent and *arr service routes through this container. Fix Gluetun before starting anything downstream.

---

## Phase 6 — Media automation

### Internal hostnames

qBittorrent, Sonarr, Radarr, and Prowlarr share Gluetun's network namespace (`network_mode: service:gluetun`). Services inside that namespace reach each other over `localhost`. Services outside it reach them via the `gluetun` container name.

| Caller | Target | Hostname |
| --- | --- | --- |
| Sonarr / Radarr → qBittorrent | same namespace | `localhost:8088` |
| Prowlarr → Sonarr / Radarr | same namespace | `localhost:8989` / `localhost:7878` |
| Unpackerr → Sonarr / Radarr | separate container | `gluetun:8989` / `gluetun:7878` |
| Jellyseerr → Sonarr / Radarr | separate container | `gluetun:8989` / `gluetun:7878` |

Using `bittorrent:8088` from Sonarr will fail. This is the most common misconfiguration in this layout.

### Bring-up order

1. **qBittorrent**

   ```bash
   docker compose up -d bittorrent
   docker logs bittorrent | grep -i "temporary password"
   ```

   Recent LinuxServer.io images generate a random temporary password. Log in at `http://127.0.0.1:8088`, change it immediately, then set **Options → Downloads → Default Save Path** to `/data/torrents`.

2. **Prowlarr** — `docker compose up -d prowlarr`, open `http://127.0.0.1:9696`, add indexers.

3. **Sonarr and Radarr** — `docker compose up -d sonarr radarr`. In each:
   - Add the download client: qBittorrent at `localhost:8088`.
   - Set root folders: `/data/media/tv` (Sonarr), `/data/media/movies` (Radarr).

4. **Link Prowlarr** — Settings → Apps, add Sonarr at `http://localhost:8989` and Radarr at `http://localhost:7878` with their API keys.

5. **Unpackerr** — copy the real API keys from each app's Settings → General into the secret files, then:

   ```bash
   docker compose up -d unpackerr
   ```

**Verify:** trigger one search in Sonarr. The download should appear in qBittorrent and import cleanly on completion.

An import failing with "file not found" means the mount paths disagree between qBittorrent and the *arr apps.

### Path layout

Every container mounts `${MEDIA_STACK_ROOT}/data` at `/data`, following the common convention:

| Host (Windows) | Inside containers |
| --- | --- |
| `D:\MediaStack\data\media\movies` | `/data/media/movies` |
| `D:\MediaStack\data\media\tv` | `/data/media/tv` |
| `D:\MediaStack\data\torrents` | `/data/torrents` |

A single shared mount root is what lets Sonarr and Radarr hardlink completed downloads instead of copying them. Giving any service a different root silently breaks that and doubles disk usage.

Jellyfin runs natively on Windows and uses the host paths in the left column.

---

## Phase 7 — Jellyfin on Windows

Jellyfin runs natively on the host, not in Docker.

1. Install with the official standalone installer (URL in `README.md`).
2. Complete the first-run wizard.
3. Add libraries pointing at `D:\MediaStack\data\media\movies` and `D:\MediaStack\data\media\tv` — native Windows paths, not container paths.
4. Allow inbound TCP **8096** through Windows Firewall on the private/LAN profile.

**Verify:** `http://<host-lan-ip>:8096` loads from a **different** device on the LAN.

---

## Phase 8 — Jellyseerr

```bash
docker compose up -d jellyseerr
```

Open `http://127.0.0.1:5055`. This local binding exists so first-run setup works before public DNS and TLS are confirmed.

In the setup wizard:

| Setting | Value |
| --- | --- |
| Jellyfin URL | `http://host.docker.internal:8096` |
| Sonarr | `http://gluetun:8989` |
| Radarr | `http://gluetun:7878` |

`host.docker.internal` resolves to the Windows host from inside the container and survives host IP changes, so no DHCP reservation is needed. On a plain Linux Docker engine that name does not exist — use the host LAN IP there.

**Verify:** `https://jellyseerr.yourdomain.com` loads over HTTPS from outside your network, and one end-to-end request reaches Sonarr or Radarr.

---

## Phase 9 — Post-deployment checks

**Kill-switch test.** Confirm the VPN is actually load-bearing:

```bash
docker stop gluetun
```

qBittorrent, Sonarr, Radarr, and Prowlarr should lose connectivity rather than falling back to your home connection. Restore with `docker compose up -d`.

**Confirm the public surface.** Only `jellyseerr` should be routable:

```bash
docker compose config | grep traefik.enable
```

**Optional port forwarding**, only if your provider supports it:

```bash
docker compose --profile portforward up -d
```

---

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Bind mounts fail on first `up` | `D:` not shared in Docker Desktop → Resources → File Sharing (Phase 1.2) |
| Traefik logs ACME authentication errors | Trailing newline in `secrets/cloudflare-token.secret`, or token missing `Zone → Zone → Read` |
| Traefik healthy but site unreachable | Router ports 80/443 not forwarded, or Cloudflare SSL mode not Full (strict) |
| Gluetun reports your home IP | Provider credentials or `VPN_SERVICE_PROVIDER` wrong — do not proceed |
| Sonarr cannot reach qBittorrent | Used `bittorrent:8088` instead of `localhost:8088` (shared namespace) |
| Downloads complete but never import | qBittorrent save path is not `/data/torrents`, or mount roots differ |
| Jellyseerr cannot reach Jellyfin | Used `localhost` (resolves to the container), or Windows Firewall blocks TCP 8096 |
| `bittorrent_port_forwarder` crash-loops | Provider does not support port forwarding; leave the `portforward` profile disabled |

---

## Maintenance

Infrastructure images are pinned to major versions (`traefik:v3`, `qmcgaw/gluetun:v3`, `containrrr/watchtower:1.7.1`, `golift/unpackerr:0.14`) so an unattended update cannot break Traefik CLI flags or the Gluetun environment contract.

Watchtower runs with `WATCHTOWER_LABEL_ENABLE=true` and updates only containers labelled `com.centurylinklabs.watchtower.enable=true`. Only Jellyseerr carries that label, because it is the sole publicly reachable service.

Update everything else deliberately:

```bash
docker compose pull
docker compose up -d
```

---

## Known unverified assumptions

Windows-specific behavior in this runbook — bind-mount path translation and the native Jellyfin install — has not been executed against a live Windows host. Phase 1 step 2 and Phase 4 are where problems would first surface.
