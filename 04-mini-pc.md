# 04 - Mini PC: Services Host

**Debian · Docker · Caddy · Vaultwarden · OwnTracks**

---

## Overview

This document covers the complete setup of the mini PC as the homelab services host. The machine runs Debian Stable with Docker for container management and Caddy as a reverse proxy with automatic TLS certificates. Two services are currently deployed: Vaultwarden (self-hosted password manager) and OwnTracks (self-hosted location sharing). Additional services (Nextcloud, Jellyfin, Immich) are planned pending NAS completion.

---

## Hardware

| Component | Spec |
|---|---|
| CPU | Intel N150 |
| RAM | 12GB DDR4 |
| Storage | 512GB internal SSD |
| OS | Debian 13 Trixie (Stable) |

---

## OS Selection

Ubuntu Server was the previous OS on this machine. Debian Stable was chosen as a replacement:

- Leaner base with no unnecessary packages
- Conservative, predictable release cycle - ideal for a server that needs to just run
- Consistent with the Pi 5, which also runs Debian
- Package staleness is a non-issue since most services run in Docker containers with their own up-to-date images

LUKS disk encryption was deliberately skipped. The machine is stationary in a physically secure home environment, and requiring a passphrase on every boot is unacceptable for an always-on headless server.

---

## OS Installation

At the software selection screen, only the following were selected:

- **SSH server** - essential for remote management
- **Standard system utilities** - base tooling

No desktop environment. The machine is fully headless.

### Post-Install Hardening

The same hardening baseline applied to the Pi 5:

- SSH key authentication (ed25519) from the T400s - password auth disabled
- `fail2ban` installed with default config
- `unattended-upgrades` configured for automatic security patches
- System fully updated via `apt update && apt upgrade`
- Hostname set to `mini-pc` via `hostnamectl`
- Static DHCP reservation set in AdGuard Home on the Flint 2

### SSH Convenience Config

An SSH config file on the T400s at `~/.ssh/config` allows connecting with `ssh mini-pc` rather than typing the full IP and username each time:

```
Host mini-pc
    HostName 192.168.8.x
    User daniel
```

`avahi-daemon` was installed on the mini PC so that `mini-pc.local` resolves automatically via mDNS on the LAN, consistent with how the Pi is accessed.

---

## Docker Installation

Docker was installed from the official Docker repository rather than the Debian repos (which ship an outdated version). Packages installed:

- `docker-ce`
- `docker-ce-cli`
- `containerd.io`
- `docker-buildx-plugin`
- `docker-compose-plugin`

User added to the `docker` group to avoid requiring `sudo` for every Docker command. Installation verified with a `hello-world` container.

All services are managed via Docker Compose with individual directories under `~/docker/` - one directory per service, each with its own `docker-compose.yml`. This keeps services clean, isolated, and independently manageable.

---

## Networking Architecture

### The Problem

Vaultwarden requires HTTPS to function. A self-signed certificate causes browser warnings and - critically - Bitwarden mobile apps often refuse to connect to self-signed certs entirely. A proper TLS setup was mandatory.

### The Solution: Caddy + Real Domain

A domain (`therofflab.net`) was registered at Porkbun. Rather than exposing services to the public internet, all subdomains resolve locally via DNS rewrites in AdGuard Home on the Flint 2. Caddy obtains real Let's Encrypt certificates via the **DNS-01 challenge** using Porkbun's API - this works without any public internet exposure or port forwarding.

Services are only accessible on the local LAN or via Tailscale. Never from the public internet.

### Shared Docker Network

A shared Docker network called `proxy` allows Caddy to communicate with service containers by name:

```bash
docker network create proxy
```

All service containers and the Caddy container are attached to this network. Services do not expose ports directly to the host - Caddy handles all inbound traffic on ports 80 and 443.

---

## Caddy Setup

### Custom Build with Porkbun Plugin

The official Caddy Docker image does not include DNS provider plugins. A custom image was built using `xcaddy` to compile in the Porkbun DNS plugin:

```dockerfile
FROM caddy:builder AS builder
RUN xcaddy build --with github.com/caddy-dns/porkbun

FROM caddy:latest
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

### Caddyfile Structure

Each service gets its own block. Porkbun API credentials are passed as environment variables:

```
vault.therofflab.net {
    tls {
        dns porkbun {
            api_key {env.PORKBUN_API_KEY}
            api_secret_key {env.PORKBUN_SECRET_API_KEY}
        }
    }
    reverse_proxy vaultwarden:80
}
```

### Issues Encountered

**Module not registered error:** The first run used the stock Caddy image instead of the custom build. Required `docker compose build --no-cache` to force a rebuild with the Porkbun plugin compiled in.

**Domain not opted in to API access:** Porkbun requires API access to be enabled at both the account level and per-domain. Both must be toggled on before the DNS-01 challenge can create TXT records.

**502 Bad Gateway:** Caddy could not reach Vaultwarden because they were on separate Docker networks. Resolved by creating the shared `proxy` network and attaching both containers to it.

---

## Vaultwarden

Vaultwarden is an open source, self-hosted implementation of the Bitwarden server - fully compatible with all official Bitwarden clients (browser extensions, mobile apps, desktop apps) while being lightweight enough to run on modest hardware. Uses SQLite by default, requiring no separate database container.

### Configuration

Key environment variables in `docker-compose.yml`:

- `SIGNUPS_ALLOWED=false` - set after initial account creation to prevent new registrations
- `ADMIN_TOKEN` - generated with `openssl rand -base64 48`, provides access to the admin panel at `/admin`

### Setup Process

1. `SIGNUPS_ALLOWED=true` temporarily set
2. Account created at `https://vault.therofflab.net`
3. `SIGNUPS_ALLOWED=false` and container restarted
4. 2FA enabled via TOTP (2FAS authenticator)
5. Passwords imported from KeePassXC using KeePass 2 (`.kdbx`) format via Tools → Import Data
6. `ADMIN_TOKEN` configured for admin panel access

### Notes

- KeePassXC exports standard `.kdbx` format, compatible with the KeePass 2 import option in Vaultwarden
- TOTP 2FA is available by default — no admin panel configuration required. Found under Settings → Security → Two-step Login in the web vault
- Bitwarden clients connect to the self-hosted instance by setting the server URL to `https://vault.therofflab.net` during login

### Remote Access

`vault.therofflab.net` resolves only via AdGuard Home's DNS rewrite - inaccessible from the public internet. Remote access works via Tailscale: the Pi 5 advertises the full `192.168.8.0/24` subnet, so any Tailscale client can reach the mini PC and resolve the domain through AdGuard Home. The mini PC itself does not need to be a Tailscale node.

---

## OwnTracks

OwnTracks is a free, open source location sharing app for iOS and Android. It pushes location data to a self-hosted OwnTracks Recorder rather than any third party - completely private, self-controlled location sharing between household members.

Exposing location data to the public internet was immediately rejected. All access is Tailscale-only, consistent with the rest of the homelab's security posture.

### Configuration

OwnTracks Recorder runs in HTTP mode - no MQTT broker required. `OTR_PORT=0` disables the built-in HTTP port since Caddy handles all inbound traffic.

`owntracks.therofflab.net` added to AdGuard Home as a DNS rewrite and to the Caddyfile with the same Porkbun TLS configuration as Vaultwarden.

### App Configuration - Android (Pixel)

- Mode: HTTP
- URL: `https://owntracks.therofflab.net/pub`
- Username: set per user
- Device ID: set per device
- No password required - Tailscale provides network-level access control

### App Configuration - iOS (iPhone)

The iOS app constructs the URL from separate fields rather than a single URL input:

- Host: `owntracks.therofflab.net` - domain only, **no** `https://` prefix
- TLS toggle: ON
- Port: `443`
- URL: `/pub`

Including `https://` in the Host field causes a malformed URL error (`NSURLErrorDomain -1002`).

### Friend Visibility

The OwnTracks Recorder web UI at `owntracks.therofflab.net` shows all users on a map. In-app friend visibility requires additional recorder configuration - deemed unnecessary since the web UI is sufficient and bookmarked on both phones.

### Tailscale Requirement

Since `owntracks.therofflab.net` only resolves locally, both phones must be on the Tailscale tailnet for OwnTracks to function away from home.

---

## Adding Future Services

Every new service follows the same pattern:

1. Create `~/docker/servicename/` directory with `docker-compose.yml`
2. Add service to the `proxy` Docker network
3. Add a DNS rewrite in AdGuard Home: `subdomain.therofflab.net` → mini PC LAN IP
4. Add a block in the Caddyfile with TLS and `reverse_proxy` directive
5. Restart Caddy: `docker compose up -d` from `~/docker/caddy/`
6. Bring up the service: `docker compose up -d` from its directory

---

## Key Lessons Learned

### Docker Networking

- Containers in separate Compose stacks cannot communicate by name unless they share a Docker network - always attach services to a shared external network when using a reverse proxy
- Never expose service ports directly to the host when Caddy is handling traffic on 80/443
- `docker compose build --no-cache` is required when a Dockerfile changes to avoid stale cached layers

### Caddy & TLS

- DNS-01 challenge via Porkbun API is the correct approach for local-only services - no public exposure required
- Porkbun API access must be enabled at both the account level and per-domain
- DNS provider plugins must be compiled into Caddy - they are not included in the official image

### OwnTracks iOS vs. Android

- iOS constructs the URL from separate Host/Port/TLS/URL fields - do not include `https://` in the Host field
- Android takes a single full URL - include the complete `https://domain/pub`
- The `/pub` endpoint is required for OwnTracks Recorder in HTTP mode

### Tailscale & Local Services

- The mini PC does not need to be a Tailscale node - the Pi's subnet router advertising `192.168.8.0/24` provides full remote access to all LAN devices
- Adding the mini PC to the tailnet caused SSH to break; removing it and relying on subnet routing resolved the issue cleanly
- All services are inherently Tailscale-only since their domains resolve only via AdGuard Home's local DNS rewrites

---

## Current State

| Component | Status |
|---|---|
| OS | Debian 13 Trixie (Stable), hardened |
| Docker | Installed, `daniel` in docker group |
| Caddy | Running, custom build with Porkbun plugin, certs active |
| Vaultwarden | Running at `vault.therofflab.net`, 2FA enabled, passwords imported |
| OwnTracks | Running at `owntracks.therofflab.net`, both phones reporting |
| Tailscale | Not installed - subnet routing via Pi handles remote access |
| Backups | Planned - Docker volumes to NAS once NAS is running |

---

## Planned Next Steps

- NAS setup (OpenMediaVault, WD Red Plus, USB 3.0 enclosure)
- Automated backups of Docker volumes from mini PC to NAS
- Deploy Nextcloud, Jellyfin, and Immich once NAS storage is available

---

## Current Caddyfile

```
vault.therofflab.net {
    tls {
        dns porkbun {
            api_key {env.PORKBUN_API_KEY}
            api_secret_key {env.PORKBUN_SECRET_API_KEY}
        }
    }
    reverse_proxy vaultwarden:80
}

owntracks.therofflab.net {
    tls {
        dns porkbun {
            api_key {env.PORKBUN_API_KEY}
            api_secret_key {env.PORKBUN_SECRET_API_KEY}
        }
    }
    reverse_proxy owntracks:8083
}
```

---

## Technologies & Skills Demonstrated

`Debian` `Docker` `Docker Compose` `Caddy` `Reverse Proxy` `TLS` `Let's Encrypt` `DNS-01 Challenge` `Vaultwarden` `Bitwarden` `OwnTracks` `Self-Hosting` `Linux Hardening` `SSH` `Networking` `mDNS`
