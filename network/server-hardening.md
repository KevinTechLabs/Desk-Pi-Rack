# Server Hardening

> Example addressing only. Usernames, keys and tokens are not published.

## Home lab server (MINISFORUM X1 Lite-255, Ubuntu Server)

### Host firewall (ufw)

| Rule | From | Purpose |
|---|---|---|
| OpenSSH | anywhere | SSH (key-only login) |
| 8088/tcp | anywhere | Sentinel dashboard |
| 1514-1515/tcp | Servers and Trusted subnets | Wazuh agent → manager |
| 5140/udp | pfSense only | Firewall and DHCP logs into Sentinel |
| any on `tailscale0` | Tailscale | Remote access |

The pfSense zones already stop IoT, Lab and Guest from reaching the server at all. ufw is the second layer.

### SSH and brute-force protection

- SSH keys only (password authentication disabled), root login disabled.
- fail2ban on sshd. Home zones are whitelisted so a mistyped username from your own desktop can't lock it out:

```ini
# /etc/fail2ban/jail.d/00-home.local
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 10.20.1.0/24 10.20.10.0/24 100.64.0.0/10
```

IoT, Lab and Guest are deliberately **not** whitelisted.

### Docker and published ports

Docker publishes container ports through its own iptables chains, **bypassing ufw**. What's running and how it's exposed:

| Container | Port | Exposure |
|---|---|---|
| Uptime Kuma | 3001 | Servers + Trusted (by pfSense zones) |
| CI/CD app | 8080 | Servers + Trusted (by pfSense zones) |
| Portainer | 9443 | **Trusted only** (DOCKER-USER rule below). Edge-agent port 8000 removed |

Portainer controls every container (root-equivalent), so it gets an extra restriction. It was recreated without port 8000, reusing the `portainer_data` volume so settings were kept:

```sh
docker stop portainer && docker rm portainer
docker run -d --name portainer --restart=always -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:lts
```

The ufw-managed DOCKER-USER rule (appended to `/etc/ufw/after.rules`, survives reboots):

```text
*filter
:DOCKER-USER - [0:0]
-A DOCKER-USER -i enp1s0 -p tcp -m conntrack --ctorigdstport 9443 --ctstate NEW -s 10.20.10.0/24 -j RETURN
-A DOCKER-USER -i enp1s0 -p tcp -m conntrack --ctorigdstport 9443 --ctstate NEW -j DROP
COMMIT
```

Tailscale traffic arrives on `tailscale0`, so it isn't affected by this rule.

## AI server (Ubuntu)

Serves Ollama (11434) and web UIs (3000, 3100). Its ufw allowed only the Servers subnet. When the desktop moved to Trusted, a matching allow rule was added:

```sh
sudo ufw allow from 10.20.10.0/24 to any port 3100,11434 proto tcp
```

> Tip: before adding a rule, check the prompt/hostname. The AI server and the home lab server have similar terminal setups, and the first attempt landed on the wrong machine.

## Kali (Lab zone)

Isolated in VLAN 40: internet and DNS only. Keep it updated (`apt full-upgrade`), leave SSH off unless needed, and change default credentials.

## Accounts

Two-factor (authenticator app or passkeys) on email, Apple, Microsoft, GitHub, Discord and Tailscale. Tailscale **device approval** is on, so new devices can't join the tailnet until approved.
