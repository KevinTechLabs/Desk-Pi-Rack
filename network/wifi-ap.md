# GL.iNet GL-BE9300 (Flint 3): Multi-SSID VLAN Access Point

> Network names are generic and no Wi-Fi passwords are stored here.

- **Mode:** Access Point (stays in AP mode; pfSense does routing, DHCP and firewalling)
- **Firmware:** GL.iNet 4.x (tested on 4.10.1)
- **Uplink:** one of the Flint 3's **LAN ports** to switch port 5. Internally that LAN port is `switch1` port 7, the CPU is port 3, and the WAN port (`eth0`) is unused
- **Management:** DHCP from the Trusted zone (the untagged VLAN on switch port 5)

| Wi-Fi network | Bands | VLAN / zone | Notes |
|---|---|---|---|
| Home (existing SSIDs) | 2.4 / 5 / 6 GHz | 10 Trusted (untagged on the uplink) | WPA2/WPA3 |
| Home-IoT | 2.4 / 5 GHz | 30 Gaming & IoT | WPA2 |
| Home-Guest | 2.4 / 5 GHz | 50 Guest | WPA2, **client isolation** on |

The GL.iNet web UI can't map SSIDs to VLANs in AP mode, so this is done over SSH with `uci`.

## Setup script (run on the Flint 3 over SSH)

The script backs up `/etc/config/network` and `/etc/config/wireless`, and prompts for the two Wi-Fi passwords, so none are ever typed into a file or shown.

```sh
TS=$(date +%Y%m%d-%H%M%S); mkdir -p /root/vlan-backup
cp /etc/config/network  /root/vlan-backup/network.$TS
cp /etc/config/wireless /root/vlan-backup/wireless.$TS

ask(){ while :; do printf "Password for %s: " "$1"; stty -echo; read -r a; stty echo; echo
  printf "Again: "; stty -echo; read -r b; stty echo; echo
  [ "$a" = "$b" ] && [ ${#a} -ge 8 ] && break; echo "No match / too short"; done; PW="$a"; }
ask Home-IoT; IOTPW="$PW"; ask Home-Guest; GPW="$PW"

for V in 30 50; do
  # switch1: carry the VLAN tagged between the CPU (3) and the uplink port (7)
  uci set network.sw_vlan$V=switch_vlan
  uci set network.sw_vlan$V.device='switch1'
  uci set network.sw_vlan$V.vlan="$V"; uci set network.sw_vlan$V.vid="$V"
  uci set network.sw_vlan$V.ports='3t 7t'
  # a bridge for the VLAN, and an interface with no IP (pure layer 2)
  uci set network.br_vlan$V=device
  uci set network.br_vlan$V.name="br-vlan$V"; uci set network.br_vlan$V.type='bridge'
  uci add_list network.br_vlan$V.ports="eth1.$V"
  uci set network.vlan$V=interface
  uci set network.vlan$V.device="br-vlan$V"; uci set network.vlan$V.proto='none'
done

mk(){ uci set wireless.$1=wifi-iface; uci set wireless.$1.device="$2"; uci set wireless.$1.mode='ap'
  uci set wireless.$1.ifname="$3"; uci set wireless.$1.ssid="$4"; uci set wireless.$1.network="$5"
  uci set wireless.$1.encryption='psk2+ccmp'; uci set wireless.$1.key="$6"
  uci set wireless.$1.isolate="$7"; uci set wireless.$1.disabled='0'; }
mk home_iot2g   wifi0 wlan05 Home-IoT   vlan30 "$IOTPW" 0
mk home_iot5g   wifi1 wlan15 Home-IoT   vlan30 "$IOTPW" 0
mk home_guest2g wifi0 wlan06 Home-Guest vlan50 "$GPW"   1
mk home_guest5g wifi1 wlan16 Home-Guest vlan50 "$GPW"   1

# bridged Wi-Fi traffic passes through the firewall on this platform: give the VLANs a zone
uci set firewall.vlanbr=zone; uci set firewall.vlanbr.name='vlanbr'
uci add_list firewall.vlanbr.network='vlan30'; uci add_list firewall.vlanbr.network='vlan50'
uci set firewall.vlanbr.input='REJECT'; uci set firewall.vlanbr.output='ACCEPT'; uci set firewall.vlanbr.forward='ACCEPT'

uci commit network; uci commit wireless; uci commit firewall
/etc/init.d/network reload; /etc/init.d/firewall reload
wifi down; sleep 3; wifi up
```

Then on the switch: port 5 untagged in VLAN 10 (PVID 10), tagged in 30 and 50, not a member of VLAN 1.

## Three gotchas found along the way

1. **New SSIDs don't appear after `wifi reload`.** On this Qualcomm platform, new virtual interfaces need a full `wifi down; wifi up`.
2. **Give each new SSID an explicit `ifname` (`wlan05`, `wlan15`, ...).** Auto-named `athXX` interfaces joined the bridge but didn't pass client traffic.
3. **Clients authenticate but never get an address.** Bridged traffic between Wi-Fi and the uplink passes through the router's firewall, so VLAN interfaces outside any firewall zone hit the default *reject*. The `vlanbr` zone (forward ACCEPT) fixes it. The AP has no IP on those VLANs, so nothing on them can reach the AP itself.

Useful checks:

```sh
iwinfo | grep ESSID                                   # SSIDs broadcasting
for b in br-vlan30 br-vlan50; do echo "$b: $(ls /sys/class/net/$b/brif)"; done
bridge fdb show br br-vlan30 | grep -v permanent      # clients seen on the IoT bridge
udhcpc -i br-vlan30 -n -q -s /bin/true                # can the AP itself get a VLAN 30 lease upstream?
```

## Other hardening

- **WPS off** on all SSIDs (`wps_pbc='0'`, `wps_pbc_enable='0'`). New devices type the password instead.
- GL.iNet's own built-in Guest and IoT networks stay **disabled**. Their built-in IoT profile uses its own static subnet and would conflict with the pfSense zones.
- GoodCloud remote management off unless needed. Keep the firmware current and use a strong admin password.

## Undo

```sh
cp $(ls -t /root/vlan-backup/network.*  | head -1) /etc/config/network
cp $(ls -t /root/vlan-backup/wireless.* | head -1) /etc/config/wireless
uci -q delete firewall.vlanbr; uci commit firewall
/etc/init.d/network reload; /etc/init.d/firewall reload; wifi down; wifi up
```
