# 01 — Home Network Infrastructure

**AT&T Fiber · GL.iNet Flint 2 · TP-Link EAP720**

---

## Overview

This document covers the full build-out of the home network layer - the foundation everything else in the homelab sits on. The goal was clean single-NAT routing, network-wide DNS control, and seamless whole-home WiFi coverage, replacing 12 years of Comcast service.

This is deliberately scoped to the physical network layer only. DNS configuration (AdGuard Home, Unbound, Tailscale) is covered in [02-pi.md](./02-pi.md).

---

## Hardware

| Role | Device |
|---|---|
| ISP Gateway | AT&T BGW320-505 (XGS-PON, cannot be removed) |
| Router | GL.iNet Flint 2 (GL-MT6000) - MediaTek quad-core, WiFi 6, 2× 2.5G + 4× 1G |
| Wireless AP | TP-Link EAP720 - WiFi 6, ethernet backhaul, AP-only mode |
| ISP | AT&T Fiber - symmetrical 1 Gbps, XGS-PON infrastructure, $47/month |

---

## Network Architecture

```
AT&T BGW320-505 (IP Passthrough)
    └── GL.iNet Flint 2 (router, DHCP, DNS, downstairs WiFi)
            ├── TP-Link EAP720 (upstairs WAP, 2.5G backhaul)
            ├── Raspberry Pi 5 (DNS + monitoring)
            ├── Mini PC (services host)
            └── Lenovo T400s (SSH management terminal)
```

**Port allocation on the Flint 2:**

- WAN (2.5G) → AT&T gateway port 1 (blue 5Gb port)
- LAN (2.5G) → EAP720 WAP
- LAN (1G) × 3 → Pi 5, Mini PC, T400s

---

## Key Concepts

### IP Passthrough vs. Double NAT

AT&T requires use of their gateway but supports IP Passthrough mode - the gateway hands the public IP directly to the downstream router, eliminating double NAT. Without passthrough, both devices perform NAT independently, causing unreliable port forwarding and degraded performance for any services exposed externally.

Passthrough is configured under **Firewall > IP Passthrough** on the gateway interface (`192.168.1.254`), by entering the WAN MAC address of the downstream router. Verification is straightforward: the Flint 2's WAN IP should show a public address, not a `192.168.x.x` private address.

### WAP vs. Wireless Extender

A wireless extender repeats the WiFi signal over the air, cutting available bandwidth roughly in half per hop and creating a technically separate network that causes roaming issues between floors. A WAP fed by ethernet (wired backhaul) is a true extension of the same network - no bandwidth penalty, same SSID, same DHCP pool, seamless roaming based on signal strength.

### DNS Override Enforcement

Many devices and applications use hardcoded DNS servers (commonly `8.8.8.8`) and will ignore whatever DNS server DHCP hands them. The Flint 2 has an **Override DNS settings for all clients** toggle that intercepts all outbound DNS traffic on port 53 and forces it through the router's configured upstream resolver - regardless of what the client requests. This is essential for enforcing network-wide DNS policy.

### IPv6

IPv6 is currently disabled on this network. If IPv6 is active but the upstream DNS resolver only listens on IPv4, devices can bypass DNS policy entirely via IPv6 DNS. Disabling IPv6 eliminates this class of leak. Re-enabling IPv6 with proper dual-stack DNS handling is planned for a future phase.

### Fiber Optic Basics

Fiber transmits data by pulsing light through glass strands - on/off states representing binary data, analogous to electrical signaling on copper. Key advantages over copper: no electromagnetic interference, lower signal degradation over distance, and support for wavelength division multiplexing (carrying multiple data streams on different light frequencies simultaneously). AT&T's local infrastructure uses XGS-PON, capable of 10 Gbps symmetric, versus older GPON limited to ~2.5G down / 1.25G up.

### Device Roaming Behavior

Different devices have different roaming aggressiveness - the RSSI threshold at which they decide to switch access points. Android in particular tends to hold connections longer before switching. This is normal. The `BSSID` field in WiFi settings shows the MAC address of the specific radio a device is currently associated with, which allows definitive confirmation of which AP is in use at any given time.

---

## Setup Process

**Phase 1 - Flint 2 initial config**
Connect to Flint 2 LAN via ethernet. Access `192.168.8.1`. Set admin password, SSID, WiFi password. Disable IPv6.

**Phase 2 - Note WAN MAC**
Record the Flint 2's WAN MAC address - needed for passthrough config on the gateway.

**Phase 3 - AT&T IP Passthrough**
Access gateway at `192.168.1.254` > Firewall > IP Passthrough. Set Allocation Mode: Passthrough, Mode: DHCPS-fixed. Enter Flint 2 WAN MAC in the Manual Entry field. Save.

**Phase 4 - Connect Flint 2 to gateway**
Ethernet from AT&T gateway port 1 (blue 5Gb port) to Flint 2 WAN port. Verify Flint 2 WAN IP is a public address.

**Phase 5 - Verify internet**
Connect a device to the Flint 2, confirm internet access and run a speed test.

**Phase 6 - EAP720 WAP setup**
Connect EAP720 to a Flint 2 LAN port. Access via DHCP-assigned IP. Configure same SSID and password as Flint 2. EAP720 operates in access point mode only by design - no routing, no separate DHCP.

**Phase 7 - AT&T gateway WiFi**
The BGW320-505 does expose a clean toggle to disable its WiFi radios in the standard UI. These were disabled to allow the Flint 2 to handle WiFi.

**Phase 8 - Reserve EAP720 IP**
Set a DHCP reservation on the Flint 2 for the EAP720's MAC address to ensure a consistent IP for management access.

---

## Results

| Metric | Result |
|---|---|
| WAN IP type | Public (passthrough confirmed, no double NAT) |
| Speed - far end of house, no WAP | ~340 Mbps down / 200 Mbps up |
| Speed - upstairs with EAP720 | ~550 Mbps down / 550 Mbps up |
| Network topology | Single seamless network, one SSID, one DHCP pool |
| DNS enforcement | Working - hardcoded DNS bypass blocked |
| Monthly cost | $47/month (down from $160 with Comcast) |

---

## Technologies & Skills Demonstrated

`Networking` `IP Passthrough` `NAT` `DHCP` `DNS` `WiFi 6` `OpenWrt` `GL.iNet` `TP-Link EAP` `Fiber Optic` `XGS-PON` `Network Architecture`
