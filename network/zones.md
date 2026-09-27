# Network Zones (VLANs)

> Example addressing only. Real subnets and host addresses are not published.

## Why zones

Before the split, every device shared one flat network, so a compromised console, smart device or guest phone could talk directly to the servers and the desktop. Now each device class lives in its own VLAN, and pfSense decides what may cross between them. A compromised device in IoT, Lab or Guest can reach the internet and nothing else.

## Zone plan

| Zone | VLAN | Interface (pfSense) | Gateway | DHCP range | DNS handed out |
|---|---|---|---|---|---|
| Servers (main LAN) | 1, untagged | `LAN` (igb0) | 10.20.1.1 | .100 – .245 | Pi-hole (10.20.1.53) |
| Trusted | 10 | `TRUSTED` (igb0.10) | 10.20.10.1 | .100 – .200 | Pi-hole |
| Gaming & IoT | 30 | `IOT` (igb0.30) | 10.20.30.1 | .100 – .200 | Pi-hole |
| Lab | 40 | `LAB` (igb0.40) | 10.20.40.1 | .100 – .200 | Pi-hole |
| Guest | 50 | `GUEST` (igb0.50) | 10.20.50.1 | .100 – .200 | Pi-hole |

The servers and Pi-hole keep their existing addresses on the untagged main LAN, so nothing that points at them had to change.

> **Lesson learned:** pick zone subnets that don't overlap the upstream (ISP) router's network. pfSense's WAN sits on that network (double NAT), and an overlapping zone subnet breaks routing for that zone.

## What each zone may reach

| From ↓ / To → | Internet | Pi-hole DNS | Servers | Trusted | IoT / Lab / Guest | pfSense itself |
|---|---|---|---|---|---|---|
| Trusted | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Servers | ✅ | ✅ (same LAN) | ✅ (same LAN) | ❌ | ❌ | ✅ |
| Gaming & IoT | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ (DHCP only) |
| Lab | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ (DHCP only) |
| Guest | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ (DHCP only) |

"❌" means the zone can't **start** a connection. Replies to connections that a permitted zone started always return.

## Device placement

| Device | Zone | Connected via |
|---|---|---|
| MINISFORUM X1 Lite-255 (home lab server) | Servers | Switch port 7 |
| AI server (i5-13600K / RTX 4070) | Servers | Switch port 8 |
| Raspberry Pi 5 (Pi-hole) | Servers | Switch port 4 |
| TP-Link TL-SG108E management | Servers | — |
| Desktop PC (daily driver) | Trusted | Switch port 3 |
| Phones and laptops | Trusted | "Home" Wi-Fi |
| GL.iNet Flint 3 management | Trusted | Switch port 5 (untagged 10) |
| Xbox | Gaming & IoT | Switch port 2 |
| Smart devices | Gaming & IoT | "Home-IoT" Wi-Fi |
| Beelink EQ (Kali Linux) | Lab | Switch port 1 |
| Visitors | Guest | "Home-Guest" Wi-Fi |

## Name resolution across zones

Windows and most desktops find local machines by broadcast name lookups (LLMNR/mDNS/NetBIOS), and those don't cross VLANs. Local hostnames therefore come from **Pi-hole Local DNS records** (hostname → server address), which work from every zone.

## Host firewalls on servers

Some services on the servers only allowed the old flat subnet. After moving the desktop to Trusted, each server's host firewall (ufw) needed an extra allow rule for the Trusted subnet on the ports it serves. See [server-hardening.md](server-hardening.md).
