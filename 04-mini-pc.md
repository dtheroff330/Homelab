# 04 — Mini PC: Services Host

**Debian · Docker · Caddy · Vaultwarden · degoog**

**DEPRECATED - OwnTracks**

---

## Overview

This document covers the complete setup of the mini PC as the homelab services host. The machine runs Debian Stable with Docker for container management and Caddy as a reverse proxy with automatic TLS certificates. Two services are currently deployed: Vaultwarden (self-hosted password manager) and degoog, a self hosted search engine. Additional services (Nextcloud, Jellyfin, Immich) are planned pending NAS completion.

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

## degoog

degoog (yes, degoog and not Degoog!) is a self-hosted search engine that aggregates results from various other engines, similar to that of SearXNG. While not as customizable as SearXNG, it benefits from being extremely easy to setup and maintain. It features a native plugin ecosystem made by the author of the project, as well as the ability to import other plugin repos for extended functionality. It even features the ability to use an OpenAI API key to use for AI generated "overviews" for searches, similar to that of popular search engines.

### Configuration

Changes to docker-compose.yml

- Due to the nature of my reverse proxy setup with Caddy and Porkbun, there was no need for ports: "4444:4444", this was instead replaced with networks: proxy
- An additional set of lines was added for the proxy setup to ensure the container looked for an external proxy. This took a fair bit of trial and error, as compared to Vaultwarden.
- DNS rewrite created on Adguard home, same as Vaultwarden.

### Setup Process

1. By default, all engines except for Wikipedia are selected. For privacy reasons, this was changed to only Brave Search.
2. Search suggestions and engine performance were toggled off for both privacy reasons and simplicity of UX for the other household users.
3. Plugins added: Define, Weather, Time, Math - each setup to use natural language input instead of needing to call a command.
4. Additional Engines: Startpage - chosen to be able to pull "close to Google" results without the legal gray area of doing so. Pairs well with Brave Search.
5. Additional Image Search: DuckDuckGo images - chosen for privacy reasons over the default Google/Bing options.
6. Intentional Exclusions: No search history - this was done to ensure household privacy, so one resident cannot see the searches of another.

### Notes

- Some engines, namely large "name-brand" engines such as Google or Bing tend to have poor results. Conversely, smaller engines (such as the ones in use) provide much more accurate results, the inverse of regular behavior.
- At times, an engine may fail to pull results. You are able to quickly retry in case of failure from the search results page.
- Rate limits are able to be applied if you have this service open to the internet for others outside of your LAN to use, although this is unnecessary in a LAN context.
- **Each user is able to change settings independently:** this allows one user to use an entirely different set of plugins and engines from another, depending on personal preference. Arguably the best feature!

### Remote Access

`search.therofflab.net` resolves only via Adguard Home's DNS rewrite, exactly the same as Vaultwarden. Remote access is functional with tailscale, although slightly slower than on the LAN as expected. Once again, the mini pc does not need to be on the tailnet for this to function.

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
| degoog | Running at `search.therofflab.net`, End-user customizable, no AI summaries implemented as of yet |
| Tailscale | Not installed - subnet routing via Pi handles remote access |
| Backups | Planned - Docker volumes to NAS once NAS is running |

---

## Planned Next Steps

- NAS setup (OpenMediaVault, WD Red Plus, USB 3.0 enclosure)
- Automated backups of Docker volumes from mini PC to NAS
- Deploy Nextcloud, Jellyfin, and Immich once NAS storage is available

---

## Deprecations & Explanations

- **OwnTracks** -> deprecated due to extremely high Android battery usage. This issue was not present on iOS, and no amount of configuration seemed to resolve the issue.

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

```

---

## Technologies & Skills Demonstrated

`Debian` `Docker` `Docker Compose` `Caddy` `Reverse Proxy` `TLS` `Let's Encrypt` `DNS-01 Challenge` `Vaultwarden` `Bitwarden` `degoog` `Self-Hosting` `Linux Hardening` `SSH` `Networking` `mDNS`
