# Homelab

A self-hosted home network infrastructure built on repurposed and low-power hardware. This repository documents the design, configuration, and outcomes of each component, including what worked, what didn't, and why decisions were made.

All systems run Linux. All services are self-hosted and open source where possible.

---

## Architecture Overview

```
Internet
  └── AT&T BGW320-505 (IP Passthrough)
        └── GL.iNet Flint 2 (Router, Firewall, AdGuard Home)
              ├── TP-Link EAP720 (Wi-Fi Access Point)
              ├── Raspberry Pi 5 (DNS, Monitoring, Tailscale)
              ├── Mini PC (Docker services)
              └── T400s (SSH management terminal)
```

**DNS chain:** Device → AdGuard Home (Flint 2, port 53) → Unbound (Pi 5, port 5335) → Root servers

**Remote access:** Tailscale subnet router on Pi 5, advertising `192.168.8.0/24`

---

## Devices

| Device | Role | OS |
|---|---|---|
| GL.iNet Flint 2 | Router, firewall, AdGuard Home | Stock firmware |
| Raspberry Pi 5 | Unbound recursive DNS, Tailscale subnet router, Prometheus/Grafana monitoring | Raspbian Lite |
| Mini PC | Docker Compose services host | Debian 13 Trixie |
| Lenovo T400s | SSH management terminal | Debian 13 Trixie + XFCE |
| NAS (Ryzen 3 3200G desktop) | Network-attached storage: in progress | OpenMediaVault (planned) |

---

## Running Services

| Service | Host | Purpose |
|---|---|---|
| AdGuard Home | Flint 2 | Network-wide DNS filtering |
| Unbound | Pi 5 | Recursive DNS resolver |
| Prometheus + Grafana | Pi 5 | Infrastructure monitoring |
| Tailscale | Pi 5 | Remote LAN Access |
| Vaultwarden | Mini PC | Self-hosted password manager |
| Caddy | Mini PC | Reverse proxy with DNS-01 TLS (Porkbun) |

---

## Security Posture

Consistent across all Linux hosts:

- SSH key-only authentication (ed25519)
- `fail2ban` (5 retries / 10 min window / 1 hr ban)
- `ufw` firewall
- `unattended-upgrades` for automatic security patches
- LUKS full-disk encryption on T400s
- Real TLS via Caddy + Porkbun DNS-01 challenge (no self-signed certificates)

---

## Upcoming

- **NAS:** Ryzen 3 3200G desktop repurposed with OpenMediaVault; 2TB WD Red Plus for storage; OS on SATA SSD. Will host Nextcloud, Jellyfin, and Immich.
- **IPv6:** Planned re-enablement across Unbound, AdGuard Home, and Flint 2 DHCPv6. Originally disabled due to lack of necessity on home network.
- **Flipper Zero:** NFC exploration and BadUSB payload testing on a dedicated test bed.

---

## Documentation

- [01-network.md](01-network.md) - AT&T BGW320-505, GL.iNet Flint 2, TP-Link EAP720
- [02-pi.md](02-pi.md) - Raspberry Pi 5: Unbound, Prometheus, Grafana, unbound_exporter
- [03-tailscale.md](03-tailscale.md) - Tailscale subnet router, remote access, DNS integration
- [04-mini-pc.md](04-mini-pc.md) - Docker Compose, Vaultwarden, Caddy
- [05-t400s.md](05-t400s.md) - SSH management terminal, security hardening, hardware repurposing
- [06-nas.md](06-nas.md) - OpenMediaVault, RAID, Data Sovereignty

---

## Technologies

`Debian` `Linux` `Docker` `Docker Compose` `Caddy` `Tailscale` `Unbound` `AdGuard Home` `Prometheus` `Grafana` `Vaultwarden` `SSH` `ufw` `fail2ban` `LUKS` `DNS` `DHCP` `Porkbun DNS-01` `ed25519` `OpenMediaVault`
