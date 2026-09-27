# TP-Link TL-SG108E (802.1Q VLANs)

- **Model:** TL-SG108E Easy Smart, hardware v6, 8 × Gigabit
- **Management:** web UI on the Servers zone (untagged VLAN 1); admin password changed from the default
- **Note:** on the Easy Smart web UI, open pages from the main page's left menu. Opening a sub-page by its direct URL loads without its scripts and can apply settings wrongly (for example, a PVID to every port).

## Port map

| Port | Device | Untagged (PVID) | Tagged |
|---|---|---|---|
| 1 | Beelink EQ (Kali) | 40 | — |
| 2 | Xbox | 30 | — |
| 3 | Desktop PC | 10 | — |
| 4 | Raspberry Pi 5 (Pi-hole) | 1 | — |
| 5 | GL.iNet Flint 3 (AP) | 10 | 30, 50 |
| 6 | pfSense (igb0, uplink) | 1 | 10, 30, 40, 50 |
| 7 | MINISFORUM home lab server | 1 | — |
| 8 | AI server | 1 | — |

## VLAN membership table

| VLAN | Name | Member ports | Tagged | Untagged |
|---|---|---|---|---|
| 1 | Default | 4, 6, 7, 8 | — | 4, 6, 7, 8 |
| 10 | TRUSTED | 3, 5, 6 | 6 | 3, 5 |
| 30 | IOT | 2, 5, 6 | 5, 6 | 2 |
| 40 | LAB | 1, 6 | 6 | 1 |
| 50 | GUEST | 5, 6 | 5, 6 | — |

## Moving a device into a zone (the order that avoids an outage)

1. **802.1Q VLAN** page: make the device's port **Untagged** in the target VLAN (keep the pfSense port Tagged there).
2. Same page: make that port **Not Member** of VLAN 1.
3. **802.1Q PVID Setting** page: set that port's PVID to the target VLAN.
4. Renew DHCP on the device (unplug and replug the cable, or on NetworkManager: `nmcli networking off && nmcli networking on`).

Enabling 802.1Q the first time shows a warning that port-based and MTU VLAN will be disabled. That's expected, since those modes aren't used.

## Identifying which device is on which port

The Easy Smart UI has no MAC table. Instead, send a burst of pings from pfSense to one device (`ping -c 3000 -i 0.002 <ip>`) and compare each port's TX packet counter before and after. The device's port jumps by about 3,000. Windows machines drop ICMP but still show the jump on TX.
