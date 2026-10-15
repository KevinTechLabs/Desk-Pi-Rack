# Vault Zone (Vaultwarden on a Dedicated Raspberry Pi 5)

> Example addressing only (`10.20.0.0/16`). Real subnets, tailnet name, keys and backup passwords are not published.

The password manager runs on its **own Raspberry Pi 5**, on its **own pfSense port**, in its **own zone**. No device on the home network can reach it. The only way in is Tailscale, from personal devices.

## Why a separate box and zone

Vaultwarden used to run as a container on the AI server, next to Ollama, Open WebUI, the CI/CD stack, Wazuh, Portainer and a NexusLab agent. A bug in any of those, or in the GPU drivers or anything the AI stack downloads, would have put an attacker on the same machine as the vault. The AI server also sits in the Servers zone, next to Pi-hole, which every restricted zone can reach.

Bitwarden clients encrypt the vault before it reaches the server, so a compromised server doesn't hand over passwords directly. The realistic risks are a **tampered web vault that captures the master password** and **offline cracking of a stolen database**. Both are much harder when nothing else runs on the box and nothing can connect to it.

| Before | After |
|---|---|
| Container on the AI server (Servers zone) | Dedicated Raspberry Pi 5, nothing else installed |
| Reachable from the Servers zone and every Trusted device | Reachable only over Tailscale, from personal devices |
| Shared host with ~20 other containers | One container |
| Served through Caddy on a non-standard port | `tailscale serve` HTTPS, Vaultwarden bound to `127.0.0.1` |

## Hardware and placement

| | |
|---|---|
| Board | Raspberry Pi 5 |
| Boot drive | WD Elements 2 TB USB drive (blue USB 3 port, no SD card) |
| OS | Raspberry Pi OS Lite (64-bit, Debian Trixie) |
| Network | Directly on pfSense **igb2**, the J1900's third port. Not on the switch, so no VLAN tagging is involved |
| Hostname | `vault` |
| Address | `10.20.60.10` (static DHCP mapping) |

Using a spare firewall port means the switch didn't need a free port. It also means the only thing between the vault and the rest of the network is pfSense itself.

## Access model

```text
  Phone / laptop / desktop ──Tailscale (WireGuard)──► vault ──tailscale serve :443──► 127.0.0.1:8080 Vaultwarden
        (personal devices)                               │
                                                         ├──► Pi-hole :53         (DNS only)
                                                         └──► internet            (updates, Tailscale, backup upload)

  Servers, IoT, Lab, Guest, Trusted LAN ──✕──► vault     (pfSense + Tailscale policy + ufw)
  vault ──✕──► any internal network                      (pfSense)
```

Three independent layers have to fail before anything on the network can reach the vault:

1. **pfSense:** VAULT can't start connections to any internal network, and Trusted is blocked from VAULT.
2. **Tailscale policy:** servers are tagged `tag:server`, and no grant lets them reach `tag:vault`.
3. **ufw on the Pi:** default deny inbound, and only `tailscale0` is allowed.

Vaultwarden itself listens on `127.0.0.1:8080` only, so even with the firewall off it isn't on the LAN.

## pfSense

### Interface

| Setting | Value |
|---|---|
| Interface | `igb2`, description `VAULT` |
| IPv4 | Static `10.20.60.1/24` (**/24, not /32**. With /32 there is no subnet, so pfSense offers no DHCP tab) |
| Upstream gateway | None |
| DHCP | `.100` – `.200`, DNS server = Pi-hole only, static mapping for the Pi at `.10` |

### VAULT rules (outbound from the Pi)

| # | Action | Proto | Destination | Port | Purpose |
|---|---|---|---|---|---|
| 1 | Block | TCP/UDP | any | 853 | No DNS-over-TLS around Pi-hole |
| 2 | Pass | TCP/UDP | Pi-hole `10.20.1.53` | 53 | DNS |
| 3 | Block (log) | any | `PRIVATE_NETS` | any | No other zones |
| 4 | Block (log) | any | This Firewall | any | No router admin |
| 5 | Pass | any | any | any | Internet |

Rule 2 has to sit above rule 3, because Pi-hole's address is inside `PRIVATE_NETS`.

### TRUSTED rule (inbound to the vault)

A block rule at the **top** of the TRUSTED tab: source TRUSTED net, destination VAULT net, logged, description *Vault: Tailscale only*. Trusted otherwise reaches every zone, so this is the one exception.

### Also added

- Forced-DNS NAT rule for VAULT (anything on port 53 that isn't Pi-hole is redirected to Pi-hole).
- VAULT added to pfBlockerNG's outbound interfaces.

## Tailscale

The vault joins the tailnet with a tag, so it isn't tied to a user and its key doesn't expire:

```sh
sudo tailscale up --hostname vault --advertise-tags=tag:vault
```

Policy file (relevant parts):

```json
{
  "tagOwners": {
    "tag:vault":  ["autogroup:admin"],
    "tag:server": ["autogroup:admin"],
  },
  "grants": [
    // personal (untagged) devices can reach everything, including the vault
    {"src": ["autogroup:member"], "dst": ["*"],          "ip": ["*"]},
    // servers can reach each other, and nothing grants them the vault
    {"src": ["tag:server"],       "dst": ["tag:server"], "ip": ["*"]},
  ],
}
```

The home lab server, AI servers, Pi-hole and the Kali box are tagged `tag:server`. Phones, laptops and the desktop stay untagged. Device approval and 2FA are on for the Tailscale account, because it's now the front door to the vault.

`tailscale status` on the Pi may show connections as relayed (DERP) instead of direct. That's expected: pfSense blocks the vault from talking directly to other internal networks, so peers on the LAN can't set up a direct path. Traffic is still end-to-end encrypted.

## The Pi

| Area | Setting |
|---|---|
| SSH | Keys only; `PermitRootLogin no`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `AllowUsers` set to the admin account |
| ufw | Default deny incoming; allow in on `tailscale0` only (the temporary LAN SSH rule was removed once Tailscale worked) |
| Updates | `unattended-upgrades` enabled |
| Docker | Docker Engine, run with `sudo` only; no remote API |
| Vaultwarden | `vaultwarden/server`, `SIGNUPS_ALLOWED=false`, no `ADMIN_TOKEN` (admin page disabled), port published on `127.0.0.1:8080` only |
| HTTPS | `tailscale serve --bg 8080` (Tailscale-issued certificate, MagicDNS name) |
| Not installed | NexusLab agent, Portainer agent, or any other remote-control software |

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vault.<tailnet>.ts.net"
      SIGNUPS_ALLOWED: "false"
    volumes:
      - ./data:/data
    ports:
      - "127.0.0.1:8080:80"
```

## Backups

Every other day at 03:00 (root crontab), `/usr/local/bin/vw-backup.sh`:

1. Takes a consistent copy of the database with `sqlite3 .backup`, which is safe while Vaultwarden is running.
2. Bundles it with `rsa_key.pem`, attachments and config into `vault-YYYY-MM-DD.tgz`.
3. Keeps **30 days** locally in `/srv/vw-backups` (mode 700).
4. Uploads to Google Drive through an **rclone `crypt` remote** (file and folder names encrypted, 256-bit generated keys) and keeps **90 days** there.

```cron
0 3 */2 * * /usr/local/bin/vw-backup.sh >> /var/log/vw-backup.log 2>&1
```

- The Google Drive remote uses the **`drive.file`** scope, so the token can only see files rclone created, not the rest of the Drive.
- The rclone crypt passwords are stored **outside Vaultwarden**. A backup you can only decrypt with the thing you lost isn't a backup.
- The local copies live on the same drive as the vault, so they cover mistakes, not drive failure. Google Drive is the off-device copy.

## Monitoring without opening a door

The NexusLab agent is pull-based and can reboot the host and control containers, so it's deliberately **not** on this Pi. For health data:

- **On demand over SSH:** `vcgencmd measure_temp`, `vcgencmd get_throttled`, `docker stats`, and board power from `vcgencmd pmic_read_adc`.
- **Planned:** the Pi **pushes** metrics to a Prometheus Pushgateway, through one narrow Tailscale grant from `tag:vault` to that one port. Data only flows out, and nothing gets a way in.

## Migration from the AI server

1. Built and hardened the Pi and the VAULT zone, then tested isolation before any data moved.
2. Created `compose.yml` on the Pi **without starting it**.
3. On the AI server: `docker stop vaultwarden`, then tarred `/opt/vaultwarden/vw-data` (SQLite, no external database).
4. Copied the archive through the desktop, the only machine that could reach both at the time.
5. Restored into `~/vaultwarden/data` (checked for `db.sqlite3` and `rsa_key.pem`), then started the container.
6. Pointed the Bitwarden iPhone app at the new self-hosted URL. Existing account, master password and 2FA carried over.
7. Locked down to Tailscale only (tags, policy, ufw, Trusted block rule) and re-tested.
8. Old container left stopped for a week as a fallback. After that it's removed, along with the Caddy entry that served it and every copy of the transfer archive.

## Tests

| Test | How | Expected |
|---|---|---|
| Vault can't reach internal networks | On the Pi: ping the Trusted gateway, Pi-hole and pfSense | All fail |
| Vault DNS and internet | On the Pi: `getent hosts example.com`; `curl -I https://deb.debian.org` | Both work |
| LAN can't reach the vault | Desktop: `ping 10.20.60.10` | Times out |
| Servers can't reach the vault over Tailscale | Home lab or AI server: `curl -m5 -k https://<vault Tailscale IP>` | Times out |
| Personal devices can | Desktop: `ssh <user>@vault`; phone: Bitwarden sync | Both work |
| Backups land encrypted | `sudo rclone ls vaultbackup:` on the Pi; Google Drive web UI | Plain names via rclone, scrambled names in Drive |

## Lessons learned

1. **Start the container only after the data is restored.** One copy step failed silently (two commands pasted onto one line). `tar` errored and Vaultwarden started with an empty data folder. Nothing was lost because sign-ups were already off and nobody logged in. The fix was `docker compose down`, wiping the empty folder and restoring again.
2. **An interface address of /32 has no subnet,** so pfSense doesn't offer a DHCP server tab for it. Use /24.
3. **Rule order matters on the VAULT tab:** the Pi-hole DNS pass rule has to come before the `PRIVATE_NETS` block.
4. **Keep a way in that isn't the network.** If Tailscale ever breaks, the Pi is still reachable with a keyboard and screen (a free KVM port). Bitwarden apps keep an offline cached copy of the vault in the meantime.
