# Network-Wide Ad Blocking with Pi-hole

A self-hosted DNS sinkhole running on a Raspberry Pi 4, providing network-wide ad and tracker blocking for every device on the home network — no per-device software, no browser extensions.

Part of a 4-project home lab portfolio. See also: [garage-temp-fan-automation](https://github.com/rubiesandgold-source/garage-temp-fan-automation)

---

## Overview

| | |
|---|---|
| **Hardware** | Raspberry Pi 4 (hostname `garagepi`) |
| **OS** | Debian GNU/Linux 13 (trixie) |
| **Network** | ASUS GT-BE98 Pro (main router) + GT-AX11000 (AiMesh node, wired garage backhaul) |
| **Static IP** | `192.168.50.120` |
| **Software** | Pi-hole (FTL / gravity blocklist engine) |
| **Upstream DNS** | Cloudflare (1.1.1.1, DNSSEC) |
| **Blocklist** | StevenBlack's Unified Hosts List (~78,000 domains) |

This is the second project on `garagepi`, sharing hardware with the [garage temp/fan automation project](https://github.com/rubiesandgold-source/garage-temp-fan-automation). Pi-hole runs alongside that project's sensor polling and fan control service with no resource conflicts (Pi-hole is lightweight — a few percent memory usage on an otherwise idle Pi 4).

---

## Why Pi-hole

Ad and tracker blocking is usually solved per-device: a browser extension here, an app-specific blocker there. That approach breaks down for devices that don't support extensions — smart TVs, phones on cellular-adjacent apps, IoT devices, game consoles. Pi-hole solves this at the network's DNS layer instead: every device that gets its DNS server from the router's DHCP settings gets ad blocking automatically, with zero per-device configuration.

---

## Network Architecture

```
Internet
   │
GT-BE98 Pro (main router, 192.168.50.1)
   │  DHCP hands out DNS Server 1 = 192.168.50.120 (Pi-hole)
   │
   ├── GT-AX11000 (AiMesh node, garage, wired backhaul)
   │
   └── garagepi (192.168.50.120)
         Pi-hole DNS + gravity blocklist
         → forwards non-blocked queries to Cloudflare (1.1.1.1)
```

**Key design decision:** DNS is configured only on the main router (GT-BE98 Pro). AiMesh nodes inherit network settings from the main router automatically — configuring DNS separately on a node isn't necessary and risks creating a conflicting configuration.

---

## Installation

### 1. Pre-flight checks

Before installing, checked for the most common Pi-hole install issue: something else already listening on port 53 (commonly `systemd-resolved`'s DNS stub listener), which would conflict with Pi-hole's own DNS service.

```bash
sudo ss -tulpn | grep :53
systemctl status systemd-resolved
```

Result: nothing was listening on port 53 (only `avahi-daemon` on port 5353 for mDNS, unrelated), and this system doesn't run `systemd-resolved` at all. No conflict — clean install path.

### 2. Install

Used Pi-hole's official install script:

```bash
curl -sSL https://install.pi-hole.net | bash
```

Installer configuration choices:

| Prompt | Choice | Reasoning |
|---|---|---|
| Network interface | `eth0` | Wired connection on the AiMesh backhaul; avoided `wlan0` (WiFi) |
| Upstream DNS | Cloudflare (DNSSEC) | Fast, and a comparatively strong public stance on query log retention |
| Blocklist | StevenBlack's Unified Hosts List | Well-maintained default; ~78,000 domains loaded on first run |
| Query logging | Enabled | Powers the dashboard, query log, and troubleshooting visibility |
| Privacy level | 0 (Show everything) | Full dashboard detail — appropriate for a single-household network, not a shared/multi-tenant one |

Install completed cleanly:

```
[i] Number of gravity domains: 78188 (78188 unique domains)
[✓] Installation complete!
```

### 3. Verify the web dashboard

Confirmed the admin interface loads and shows Pi-hole as **Active** at `http://192.168.50.120/admin` before touching any network-wide settings — validating the install in isolation before making it live for the whole network.

### 4. Point the network's DHCP at Pi-hole

On the main router (GT-BE98 Pro) → **LAN → DHCP Server → DNS and WINS Server Setting**:

- **DNS Server 1:** `192.168.50.120`
- **DNS Server 2:** *(left blank — see design decision below)*
- **Advertise router's IP in addition to user-specified DNS:** **No**

**Design decision — no DNS Server 2 fallback:**
It's tempting to set a secondary DNS server (like `1.1.1.1`) as a "just in case" fallback if Pi-hole ever goes down. This was deliberately avoided: a fallback means devices can silently switch between Pi-hole and public DNS, producing inconsistent, hard-to-diagnose ad-blocking behavior (ads appearing intermittently with no clear cause). Leaving DNS Server 2 blank means a Pi-hole outage is immediately obvious (no internet, rather than quietly-degraded blocking) — a deliberate reliability-over-convenience tradeoff.

Also disabled **"Advertise router's IP in addition to user-specified DNS"** — left enabled, the router would hand out both Pi-hole's IP and its own IP as valid DNS options, letting devices bypass Pi-hole entirely by using the router's IP directly.

---

## Verification

Confirmed via the Pi-hole dashboard that:
- Total Queries and Queries Blocked began incrementing from 0 once devices renewed their DHCP leases
- The Query Log showed real client requests and blocked domains in real time

### Live results, several hours after going live

![Pi-hole Dashboard](Screenshot%202026-07-06%20153941.png)

9,906 total queries, 779 blocked (7.9%), 20 active clients — the traffic spike visible on the graph marks the point where the router's DHCP DNS setting was applied and devices began renewing their leases onto Pi-hole.

![Blocked Domains and Client Activity](Screenshot%202026-07-06%20153945.png)

The most-blocked domains are exactly the category Pi-hole is meant to catch — telemetry and ad-serving endpoints (`scribe.logs.roku.com`, `incoming.telemetry.mozilla.org`, `googleads.g.doubleclick.net`, `app-measurement.com`) — while core functionality for the same services (Netflix, YouTube, Amazon) passes through untouched, confirming the blocklist isn't overly aggressive. Multiple distinct client IPs show activity, confirming the blocking is genuinely network-wide rather than limited to a single device.

---

## Troubleshooting Notes

**Gaming/false-positive domains:** Since a household gaming PC (`INFERNO`, formerly `DayvanCowboy`) is on this network, part of the plan included checking for false-positive blocked domains after the DNS switch — some game platforms' analytics/telemetry subdomains occasionally overlap with ad/tracking blocklists. Process: check the Query Log for red/blocked entries around the time a connection issue occurs, then whitelist the specific domain if it's a legitimate game service. No reboot required for whitelist changes to take effect.

**Router DHCP client label lag:** After renaming a device on the network, the router's DHCP reservation table may continue showing the device's *old* hostname as a cached label until the device fully reconnects/renews its lease. This is cosmetic only and doesn't affect DNS routing or the reservation's function (which is keyed to the device's MAC address, not its display label).

---

## Lessons Learned

- Always check for port 53 conflicts (`systemd-resolved` or similar) before installing any local DNS service — this is the single most common Pi-hole install snag, even though it didn't apply here.
- On AiMesh networks, DNS configuration belongs on the main router only; nodes inherit it automatically.
- The "no DNS fallback" decision is a good example of choosing a *visible failure* over a *silent, inconsistent one* — a pattern worth applying elsewhere in network configuration.

---

## Next Steps

- Add supplemental blocklists (tracking/analytics-focused, beyond the default StevenBlack list)
- Set up Pi-hole's built-in update checks / periodic gravity list refresh
- Consider a second Pi-hole instance (or conditional forwarding) for redundancy, now that the household is fully dependent on it for DNS

---

## Related Projects

This is one of four planned home lab portfolio projects on the same Raspberry Pi 4 hardware:

1. [Garage Temperature & Fan Automation](https://github.com/rubiesandgold-source/garage-temp-fan-automation)
2. **Pi-hole Network-Wide DNS Ad Blocking** *(this project)*
3. *(planned)*
4. *(planned)*
