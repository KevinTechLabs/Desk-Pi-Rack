# pfSense Firewall

> Example addressing only (`10.20.0.0/16`). No credentials, keys, public IPs or config backups are stored in this repository.

- **Hardware:** Intel J1900 mini PC, 4 × Intel i210 (`igb0`–`igb3`), 4 GB RAM
- **Software:** pfSense CE 2.8.x
- **Interfaces:** `igb1` = WAN (DHCP from the upstream router), `igb0` = LAN trunk (VLAN 1 untagged + VLANs 10/30/40/50 tagged), `igb2`/`igb3` unused

## 1. Hardening the router itself

| Setting | Where | Value |
|---|---|---|
| Firmware | System → Update | Latest stable. See the certificate note in [maintenance.md](maintenance.md) |
| Admin password | System → User Manager | Long and unique, kept in a password manager |
| Web admin | System → Advanced → Admin Access | HTTPS only; anti-lockout rule kept on LAN |
| SSH | Same page | **Disabled**; SSHd Key Only = *Public Key Only*, so if it's ever enabled, passwords won't work |
| WAN reserved networks | Interfaces → WAN | **Block bogon networks** on. *Block private networks* **off**, because the WAN sits on the upstream router's private network (double NAT) |
| Inbound exposure | Firewall → Rules → WAN / NAT | No WAN pass rules, no port forwards |
| UPnP / NAT-PMP | Services | Not installed or enabled |
| Remote access | — | Tailscale only (no ports opened on pfSense) |

## 2. Alias

`PRIVATE_NETS` (Firewall → Aliases, type Network): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
It's used to say "any other internal zone, including the upstream router's network".

## 3. Rules per zone

pfSense evaluates rules top to bottom per interface, and the first match wins.

### LAN (Servers)

| # | Action | Source | Destination | Purpose |
|---|---|---|---|---|
| — | Pass | any | LAN address :443/:80 | Anti-lockout (built in) |
| 1 | Pass | LAN net | This Firewall | Servers may use the router (DNS resolver, time, admin) |
| 2 | Block | LAN net | `PRIVATE_NETS` | Servers can't start connections into other zones |
| 3 | Pass | LAN net | any | Internet |

### TRUSTED

| # | Action | Source | Destination | Purpose |
|---|---|---|---|---|
| 1 | Pass | TRUSTED net | any | Own devices reach everything |

### IOT, LAB, GUEST (identical pattern)

| # | Action | Proto | Destination | Purpose |
|---|---|---|---|---|
| 1 | Block | TCP/UDP | any :853 | No DNS-over-TLS around Pi-hole |
| 2 | Pass | TCP/UDP | Pi-hole :53 | DNS |
| 3 | Block | any | `PRIVATE_NETS` | No other zones |
| 4 | Block | any | This Firewall | No router admin or services (DHCP still works via pfSense's built-in rules) |
| 5 | Pass | any | any | Internet |

### Tested results

From the Lab zone: internet ✅, DNS ✅, home lab server ❌, pfSense admin ❌, desktop PC ❌.
From the Servers zone: internet ✅, desktop PC ❌, Xbox ❌.
From Trusted: every service on the servers ✅.

## 4. NAT: forced DNS

Firewall → NAT → Port Forward, one rule per zone (TRUSTED, IOT, LAB, GUEST):

- Protocol TCP/UDP, destination **NOT** Pi-hole, port 53
- Redirect target: Pi-hole, port 53
- Filter rule association: *Pass*

A device with hard-coded DNS (for example `8.8.8.8`) silently gets Pi-hole's answer instead. This works across VLANs because the reply passes back through pfSense, which reverses the translation. The Servers zone is deliberately left out, because Pi-hole lives there and uses pfSense as its upstream.

## 5. pfBlockerNG-devel (IP reputation blocking)

- **Mode:** IP blocking only. **DNSBL is off**, so Pi-hole stays the single DNS filter and filtering doesn't stack inside the resolver Pi-hole depends on.
- **IP settings:** inbound interface WAN; outbound interfaces LAN + all VLANs; floating rules on; de-duplication and aggregation on; kill states on update.
- **Feeds (group PRI1, action *Deny Both*, refresh every 4 h):** Abuse.ch Feodo C2, CINS Army, ET Block, ET Compromised, ISC Block, Spamhaus DROP. Pulsedive is left off.
- **Size at install:** about 13,000 addresses.
- The setup wizard was skipped on purpose (`pfblockerng_general.php?wizard=skip`), because it also enables DNSBL.

## 6. Suricata (intrusion detection)

- **Interface:** WAN, legacy mode, **Block Offenders off** for the first week (watch-only), then reviewed and switched on.
- **Rules:** ET Open, Feodo Tracker botnet C2, Abuse.ch SSL blacklist; update every 12 h with live rule swap; blocked hosts cleared after 1 h.
- **Categories enabled:** attack_response, botcc, botcc.portgrouped, ciarmy, coinminer, compromised, drop, dshield, exploit, exploit_kit, malware, mobile_malware, scan, feodotracker, sslblacklist.
- **Resources on the J1900:** about 37,000 rules, about 525 MB RAM, under 2 % CPU at idle (rule compilation briefly pegs one core at start).
- **Required:** System → Advanced → Networking → **Disable hardware checksum offload** (then reboot). Without it, Suricata logs floods of "invalid checksum" alerts and skips inspecting those packets.
- A handful of ET rules fail to load (JA3 not enabled). That's expected and harmless.

## 7. Logging

Status → System Logs → Settings → Remote Logging:
- Source address: LAN
- Remote server: the home lab server, UDP 5140 (Sentinel)
- Contents: **Firewall Events** and **DHCP Events**

The server only accepts UDP 5140 from pfSense's address (ufw).

## Recovery

- Keep a keyboard and monitor (or console cable) available. Console menu option 15 restores a recent configuration.
- The anti-lockout rule keeps the web admin reachable from LAN even if a rule change goes wrong.
- A fresh configuration backup is downloaded after every change session and stored **offline**, never in this repository.
