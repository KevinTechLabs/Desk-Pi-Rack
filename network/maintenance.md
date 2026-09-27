# Maintenance, Recovery and Lessons Learned

## Routine

| When | Task |
|---|---|
| Daily | Glance at Sentinel's Discord summary |
| Monthly | pfSense and Pi-hole updates (`pihole -up`), then download a fresh pfSense config backup and store it **offline** |
| Every few months | TL-SG108E and GL.iNet firmware |
| After any change | Re-run the isolation tests below |

## Isolation tests

| Test | How | Expected |
|---|---|---|
| Lab isolated | From the Kali box: ping the home lab server and the desktop; `curl -k https://<pfSense>` | All fail; internet works |
| IoT isolated | Phone on Home-IoT: open the Sentinel dashboard | Doesn't load; websites do |
| Guest isolated | Phone on Home-Guest: same | Doesn't load; websites do |
| Servers can't reach in | From the home lab server: ping the desktop and the Xbox | Both fail; internet works |
| Trusted works | Desktop: SSH, Sentinel, Portainer, AI web UI | All work |
| Forced DNS | Lab: `nslookup example.com 8.8.8.8` | Answers; appears in Pi-hole's Query Log |
| DoT blocked | Lab: `timeout 5 bash -c '</dev/tcp/1.1.1.1/853'` | Fails |
| Nothing open inbound | Phone on mobile data: online port scan of the public IP | All ports stealth or closed |

## Backups

- **pfSense:** Diagnostics → Backup & Restore → Download configuration. The file contains password hashes and the full rule set, so store it privately (password manager or encrypted drive). **Never commit it here.**
- **GL.iNet:** the VLAN script keeps copies in `/root/vlan-backup/` on the device.

## Recovery

- pfSense console menu option 15 restores a recent configuration.
- The anti-lockout rule keeps the pfSense web admin reachable from the main LAN.
- The switch can be factory-reset with its reset pin; with no VLANs configured it behaves as a plain switch.

## Lessons learned

1. **pfSense said "up to date" when it wasn't.** The package repository fetch was failing with *self-signed certificate in certificate chain*. The certificate chain was genuine (Sectigo → USERTrust), but the hashed trust store in `/etc/ssl/certs` held only one entry. Running `certctl rehash` fixed it, and the upgrade then showed up.
2. **Suricata "invalid checksum" floods:** disable hardware checksum offload in pfSense and reboot.
3. **The pfBlockerNG wizard also enables DNSBL.** Skip it when Pi-hole is the DNS filter.
4. **Stray blocked packets from Google/Cloudflare** after a state reset (TCP without SYN, UDP from ports 443/53) are late replies, not attacks. Sentinel ignores them (v1.5.1).
5. **Moving a PC to a new VLAN** changes its source address. Check server host firewalls, fail2ban ignore lists and Wazuh allow rules for the old subnet.
6. **Broadcast name lookups don't cross VLANs.** Use Pi-hole Local DNS records.
7. **Pick zone subnets that don't overlap the upstream router's network.**
8. **Keep a backup of the config before every change session,** and make changes in an order where each step is reversible (create VLANs in pfSense first, tag the trunk ports, then move one device at a time).
