# 06 — NAS: Centralized Data Ownership

**OpenMediaVault · mdraid · Samba · Ryzen G-Series · Data Sovereignty**

---

## Overview

This document outlines the theoretical build-out of the home NAS (Network Attached Storage). This machine is the final piece of the current homelab phase, designed to transition 10+ years of photo backups and Docker volume data from fragmented strategies to a centralized, redundant, and owned storage array.

---

## Hardware

| Component | Spec |
|---|---|
| CPU | AMD Ryzen G-Series (Integrated Vega Graphics) |
| RAM | 16GB DDR4 |
| Boot Drive | SATA SSD (Internal) |
| Storage Pool | Planned: 2x WD Red Plus (Mirrored) |
| OS | OpenMediaVault (OMV) 7 (Debian 12 Bookworm based) |

---

## OS Selection: OpenMediaVault

While TrueNAS was considered, OpenMediaVault was chosen for the following reasons:

- **Debian Base:** OMV 7 is built on Debian 12 Bookworm, ensuring stability.
- **Hardware Flexibility:** Better support for non-enterprise hardware and easier drive expansion compared to ZFS-only systems.
- **Web UI:** Provides a robust interface for managing RAID, S.M.A.R.T. monitoring, and shared folders without heavy CLI overhead for routine tasks.

---

## Planned Architecture

### Storage Strategy
To protect the 10-year photo collection, a **mirrored (RAID 1)** configuration is the baseline. This ensures that a single drive failure does not result in data loss.

### Integration with Mini PC
The mini PC will mount NAS shares via **NFS** or **SMB**. Docker volumes for Vaultwarden and other services will be backed up to the NAS on a cron schedule, moving them off the mini PC's internal SSD.

### Media Handling
The Ryzen G-series chip provides a significant advantage for **Jellyfin**. The integrated Vega graphics will be utilized for **Hardware Transcoding**, allowing the NAS to serve high-bitrate video to devices on the local network without taxing the CPU.

---

## Setup Process (Theoretical)

**Phase 1 - OMV Installation**
Install OMV 7 via the ISO. Configure the system on a dedicated SATA SSD to keep the storage pool clean.

**Phase 2 - Network Configuration**
Set a static IP (`192.168.8.x`) and disable IPv6 to maintain consistency with the current network policy.

**Phase 3 - Pool Creation**
Initialize the WD Red drives. Set up a mirrored pool and create filesystem mount points.

**Phase 4 - Service Deployment**
Enable SMB/CIFS for Windows systems and NFS for Linux systems.

---

## Technologies & Skills Demonstrated

`OpenMediaVault` `RAID` `mdraid` `Samba/NFS` `Data Redundancy` `Hardware Transcoding` `Ryzen` `Debian` `Storage Architecture` `NAS`