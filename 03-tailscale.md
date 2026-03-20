# 03 — Tailscale: Remote Access & Network-Wide DNS

**Tailscale · WireGuard · Subnet Router · AdGuard Home · Unbound**

---

## Overview

This document covers the full setup of Tailscale for remote access to the homelab. The primary goal was maintaining AdGuard Home + Unbound DNS filtering on all personal devices regardless of network location. A secondary goal was enabling secure remote access to all homelab services (Grafana, SSH, Vaultwarden, etc.) without any public internet exposure or port forwarding.

The session involved one failed approach (Tailscale on the Flint 2) and one successful approach (Tailscale on the Pi), with important lessons from both.

---

## Goals

- AdGuard Home + Unbound DNS on all personal devices, even away from home
- Remote SSH access to Pi, T400s, and other homelab devices
- Access to internal services (Grafana, AdGuard dashboard, Vaultwarden) without port forwarding
- Per-device, opt-in - other household members completely unaffected

---

## Architecture Decisions

### Tailscale vs. Headscale

Headscale (self-hosted Tailscale control plane) was considered and rejected on KISS grounds. Tailscale is already built natively into the Flint 2 firmware - adding a self-hosted control plane would introduce maintenance overhead without proportional benefit. Tailscale's free tier covers all personal use needs.

### Tailscale vs. ZeroTier

ZeroTier operates at Layer 2 (virtual Ethernet switch) - more flexible for complex multi-site setups but lacking built-in DNS management. Tailscale operates at Layer 3 (WireGuard-based), has native DNS configuration in the admin panel, and is significantly simpler to manage. For a single-person homelab focused on DNS and remote access, Tailscale is the clear choice.

### Exit Node vs. Subnet Router

A full exit node routes all internet traffic through home. A subnet router only makes LAN resources accessible over the tunnel while internet traffic routes normally. The subnet router approach was chosen - lightweight, purposeful, and sufficient for all goals. Internet traffic away from home exits via cellular directly; only DNS queries and LAN access traverse the tunnel.

---

## What Didn't Work: Tailscale on the Flint 2

The GL.iNet Flint 2 has native Tailscale support in firmware, so this was the first approach attempted. Initial setup appeared to work correctly on WiFi but failed completely on cellular - pages would not load at all when Tailscale was active on mobile data.

### Root Cause

The GL.iNet Tailscale implementation is marked **Beta** in the firmware UI. The subnet routing implementation was unreliable for external clients arriving over cellular. On WiFi, the phone is already on the `192.168.8.x` LAN and can reach AdGuard Home directly. On cellular, traffic arrives via the Tailscale tunnel and the beta firmware failed to route it correctly.

### Decision

Reverted all Tailscale configuration on the Flint 2 and moved the implementation to the Pi, where Tailscale has full official Linux support.

---

## What Worked: Tailscale on the Pi

### Installation

Installed via the official script:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Brought up as a subnet router advertising the full home LAN:

```bash
sudo tailscale up --advertise-routes=192.168.8.0/24
```

### The IP Forwarding Fix - Key Lesson

After initial setup, cellular access still failed. Diagnosis revealed IP forwarding was disabled on the Pi:

```bash
cat /proc/sys/net/ipv4/ip_forward
# returned: 0
```

IP forwarding tells the Linux kernel to pass packets between network interfaces rather than dropping them. Without it, the Pi receives Tailscale tunnel traffic destined for the LAN (e.g. AdGuard Home at `192.168.8.1`) and drops it - because the packet isn't addressed to the Pi itself. This is the same fundamental function a router performs. The Pi needed to be explicitly told to forward packets.

Enabled immediately and made permanent:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

As a note, the above command did not persist through a forced reboot (power outage). You must force them to persist via /etc/sysctl.d/. Otherwise, tailscale will use a DERP relay and slow down dramatically.

After enabling IP forwarding and reconnecting Tailscale on the iPhone, cellular access worked correctly. Confirmed at the gym on mobile data.

### DNS Configuration

In the Tailscale admin panel (`login.tailscale.com/admin/dns`):

1. Added `192.168.8.1` as a Global Nameserver (AdGuard Home on the Flint 2)
2. Enabled **Override DNS servers** - forces all Tailscale devices to use this nameserver rather than treating it as optional
3. Enabled **MagicDNS** — allows reaching devices by hostname (e.g. `rasp-pi`) rather than IP

**Full DNS chain away from home:**

```
Phone on cellular
  └── Tailscale tunnel (WireGuard)
        └── AdGuard Home (192.168.8.1, Flint 2)
              └── Unbound (192.168.8.196:5335, Pi)
                    └── Root DNS servers
```

---

## Tailnet Device Roster

| Device | Role | Notes |
|---|---|---|
| Pi (`rasp-pi`) | Subnet router | Advertises `192.168.8.0/24`, IP forwarding enabled, key expiry disabled |
| iPhone Air | Primary mobile client | `accept-routes` enabled |
| T400s (`dbT400s`) | Client | Debian, `accept-routes` enabled, key expiry disabled |
| ThinkPad P15 Gen 2 | Client | Windows, occasional travel device |

---

## Privacy Impact

### At Home (unchanged)

- DNS queries resolve via Unbound to root servers - AT&T cannot see query content
- AT&T can see destination IPs of connections (unrelated to DNS)

### Away from Home (significant improvement)

- Mobile carrier sees only encrypted WireGuard traffic to the home IP - cannot see DNS queries or destinations
- Home ISP sees the WireGuard tunnel arriving but cannot inspect its contents
- Unbound eliminates third-party DNS resolvers (Cloudflare, Google) entirely

---

## AdGuard Home: Client Attribution

During setup, per-device client attribution was explored and ultimately abandoned. Several issues made it impractical:

- iOS and Android use MAC address randomization by default, causing devices to appear with different identities on each connection
- Tailscale routes T400s DNS through the Pi as subnet router, making T400s queries appear to originate from the Pi in AdGuard Home logs
- Per-device tracking was not aligned with the actual goal - network health visibility, not household surveillance

**Resolution:** Enabled **Anonymize Client IP** in AdGuard Home settings. Query logs retain full usefulness for diagnosing blocked and allowed domains without per-device attribution.

**Grafana telemetry:** `stats.grafana.com` and `stats.grafana.org` were generating ~8,000 blocked queries per 24 hours. Whitelisted in AdGuard Home with `@@||stats.grafana.com^` and `@@||stats.grafana.org^` - self-hosted infrastructure with no privacy concern.

---

## Outstanding: IPv6

IPv6 remains disabled at the Flint 2 level. Re-enabling it was planned during this session but deferred to avoid network disruption during an active configuration session.

When IPv6 is re-enabled, the following will need to be addressed:

- **Unbound:** enable `do-ip6`, add `::0` interface, add IPv6 private address ranges (`fd00::/8`, `fe80::/10`), add IPv6 access control
- **AdGuard Home:** configure IPv6 upstream pointing to Unbound
- **Flint 2:** re-enable IPv6 and enforce DNS via DHCPv6 in the same way as IPv4

---

## Final Architecture

The Pi's role is now fully defined:

| Service | Role |
|---|---|
| `unbound` | Recursive DNS resolver, port 5335 |
| `tailscale` | Subnet router, advertises `192.168.8.0/24` |
| `prometheus` + `grafana-server` | Network and system monitoring |
| `prometheus-node-exporter` + `unbound_exporter` | Metrics collection |
| `fail2ban` | SSH brute force protection |
| `unattended-upgrades` | Automatic security patching |

Everything on the Pi is network infrastructure. The Pi is considered complete.

---

## Technologies & Skills Demonstrated

`Tailscale` `WireGuard` `VPN` `Subnet Routing` `DNS` `IP Forwarding` `Linux Networking` `AdGuard Home` `Unbound` `Network Privacy` `Systemd` `Debian` `Raspberry Pi`
