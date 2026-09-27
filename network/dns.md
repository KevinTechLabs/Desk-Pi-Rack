# DNS

> Example addressing only.

## Design

```text
 any zone ──DNS──► Pi-hole (10.20.1.53) ──► pfSense DNS Resolver (10.20.1.1) ──► root servers
                     ads / trackers            recursive, DNSSEC validated
                     blocked here              no third-party DNS provider
```

- **Pi-hole** (Raspberry Pi 5) is the only DNS server handed out by DHCP in every zone.
- **Pi-hole upstream:** only `10.20.1.1`, pfSense's resolver (Unbound). All public upstream providers are unticked.
- **pfSense DNS Resolver:** enabled, recursive (forwarding mode off), DNSSEC on, listens on all interfaces. Verified: a deliberately broken DNSSEC test domain returns SERVFAIL.
- **Pi-hole → Interface settings:** *Permit all origins*, so it answers the other VLANs. This is safe only because pfSense keeps the internet out.

## Stopping devices from bypassing Pi-hole

| Bypass attempt | Countermeasure |
|---|---|
| Hard-coded DNS server (`8.8.8.8`, `1.1.1.1`, ...) | pfSense NAT redirect: port 53 to anything but Pi-hole is rewritten to Pi-hole (Trusted, IoT, Lab, Guest) |
| DNS-over-TLS (port 853) | Blocked on IoT, Lab and Guest |
| DNS-over-HTTPS in browsers | Pi-hole's default lists include the Firefox canary domain; other DoH can't be blocked by port |

Test from a Lab device: `nslookup example.com 8.8.8.8` still answers, and the query shows up in Pi-hole's Query Log from the Lab address.

## Local names

Cross-VLAN clients can't use broadcast name lookups, so servers get **Pi-hole Local DNS records** (hostname → address). Hosts can use either the short name or IP.
