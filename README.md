# OpenWrt + NordVPN (NordLynx/WireGuard) VPN Router Guide

Route all traffic through NordVPN on a secondary router running OpenWrt, while keeping your main router untouched. Every device connected to the VPN router gets a VPN IP automatically - no per-device configuration needed.

---

## Infrastructure

```
ISP (1 Gbps)
    │
    └──► Main Router (192.168.1.1)
              │  [LAN port → Cat5e/Cat6 cable]
              │
              └──► Secondary Router WAN port
                        │
                   OpenWrt (192.168.2.1)
                   WireGuard → NordVPN
                        │
                        ├──► WiFi SSID: "YourName_VPN_USA"
                        │         └──► All WiFi devices → VPN
                        │
                        └──► LAN ports (blue)
                                  └──► All wired devices → VPN
```

> ⚠️ **Important:** Use a Cat5e or Cat6 cable between routers. A Cat5 cable will cap your speed at 100 Mbps.

---

## Hardware Used

- **Router:** Linksys WRT3200ACM
  - Marvell Armada 385 dual-core 1.6GHz with hardware AES
  - 512MB RAM, 256MB flash
  - Dual-boot flash partitions (easy recovery)
- **Firmware:** OpenWrt 25.x (uses `apk` package manager)
- **VPN:** NordVPN (NordLynx/WireGuard protocol)

### Expected Speeds
| Connection | Speed |
|---|---|
| Wired through VPN | 200-250 Mbps |
| WiFi through VPN | ~180 Mbps |
| VPN latency (US server from Israel) | ~335ms |

---

## Prerequisites

- A secondary router supported by OpenWrt (Linksys WRT3200ACM recommended)
- NordVPN subscription
- A NordVPN access token (generated at [my.nordaccount.com](https://my.nordaccount.com/dashboard/nordvpn/access-tokens/))
- A Cat5e or Cat6 ethernet cable between your main router and secondary router
- A PC with PowerShell (for getting NordVPN credentials)

---

## Phase 1: Flash OpenWrt

### Step 1 - Download OpenWrt Image

Go to [openwrt.org](https://openwrt.org) and find your device. For WRT3200ACM:
```
https://openwrt.org/toc/toc/linksys/wrt3200acm
```
Download the **factory image** (not sysupgrade):
```
openwrt-x.x.x-mvebu-cortexa9-linksys_wrt3200acm-squashfs-factory.img
```

### Step 2 - Flash via Stock Linksys UI

1. Connect your PC to a LAN port (blue) on the Linksys
2. Open browser → `192.168.1.1`
3. Login: `admin` / `admin` (or blank password)
4. Go to **Connectivity → Router Firmware Update → Manual**
5. Select the `.img` file → click **Start**
6. Wait ~3 minutes - do NOT touch anything

### Step 3 - First Access

- Browser → `192.168.1.1`
- Login: `root` / no password
- Immediately set a root password: **System → Administration**

---

## Phase 2: Change LAN IP (Avoid Conflict with Main Router)

Both routers default to `192.168.1.1` - we need to change OpenWrt's IP.

Connect your PC **directly** to a blue LAN port on the Linksys (not through main router yet).

```bash
ssh root@192.168.1.1

uci set network.lan.ipaddr='192.168.2.1'
uci commit network
reboot
```

After reboot, set your PC ethernet to:
- IP: `192.168.2.2`
- Subnet: `255.255.255.0`
- Gateway: `192.168.2.1`

Then SSH back in:
```bash
ssh root@192.168.2.1
```

---

## Phase 3: Connect to Main Router & Install Packages

### Physical Setup
```
Main Router LAN port
    │  [ethernet cable]
    └──► Linksys WAN port (yellow)

Linksys LAN port (blue) ──► Your PC
```

### Enable WiFi (so you can go wireless later)
```bash
uci set wireless.radio0.disabled='0'
uci set wireless.radio1.disabled='0'
uci commit wireless
wifi reload
```

### Install WireGuard
```bash
apk update
apk add wireguard-tools kmod-wireguard
```

> **Note:** OpenWrt 25.x uses `apk` instead of `opkg`. If you're on an older version, replace `apk add` with `opkg install`.

---

## Phase 4: Get NordVPN Credentials

### Step 1 - Generate Access Token
1. Go to [my.nordaccount.com/dashboard/nordvpn/access-tokens](https://my.nordaccount.com/dashboard/nordvpn/access-tokens/)
2. Create a new token and copy it

### Step 2 - Get Your WireGuard Private Key (PowerShell)
```powershell
$accessToken = "PASTE_YOUR_ACCESS_TOKEN_HERE"
$credentials = "token:$accessToken"
$encodedCredentials = [System.Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($credentials))
$headers = @{ "Authorization" = "Basic $encodedCredentials" }
$response = Invoke-RestMethod -Uri "https://api.nordvpn.com/v1/users/services/credentials" -Headers $headers
$response.nordlynx_private_key
```

Save the output - this is your **WireGuard private key**. It should look like base64 (e.g. `aBcDeFgH...=`), not hex.

### Step 3 - Get a US Server (PowerShell)

> **Known issue:** NordVPN's recommendations API may return servers from your local region regardless of country filter. Use a specific known hostname to get reliable US servers.

```powershell
$servers = Invoke-RestMethod -Uri "https://api.nordvpn.com/v1/servers?filters[servers_technologies][identifier]=wireguard_udp&filters[domain]=us9354.nordvpn.com"

$servers | ForEach-Object {
    $country = $_.locations[0].country.name
    $pubkey = ($_.technologies | Where-Object { $_.identifier -eq 'wireguard_udp' }).metadata | Where-Object { $_.name -eq 'public_key' } | Select-Object -ExpandProperty value
    [pscustomobject]@{
        Name      = $_.name
        IP        = $_.station
        Country   = $country
        PublicKey = $pubkey
    }
} | Where-Object { $_.Country -eq "United States" } | Format-Table -AutoSize
```

Pick any server from the list. Save the **IP** and **PublicKey**.

---

## Phase 5: Configure WireGuard on OpenWrt

SSH into the router and run all of the following:

### Step 1 - Create WireGuard Interface
```bash
uci set network.wg0=interface
uci set network.wg0.proto='wireguard'
uci set network.wg0.private_key='PASTE_YOUR_PRIVATE_KEY_HERE'
uci set network.wg0.addresses='10.5.0.2/32'
uci commit network
```

### Step 2 - Add NordVPN Peer
```bash
uci add network wireguard_wg0
uci set network.@wireguard_wg0[-1].description='NordVPN'
uci set network.@wireguard_wg0[-1].public_key='PASTE_SERVER_PUBLIC_KEY_HERE'
uci set network.@wireguard_wg0[-1].endpoint_host='PASTE_SERVER_IP_HERE'
uci set network.@wireguard_wg0[-1].endpoint_port='51820'
uci set network.@wireguard_wg0[-1].allowed_ips='0.0.0.0/0'
uci set network.@wireguard_wg0[-1].persistent_keepalive='25'
uci commit network
```

### Step 3 - Firewall Zones
```bash
uci add firewall zone
uci set firewall.@zone[-1].name='vpn'
uci set firewall.@zone[-1].input='REJECT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='REJECT'
uci set firewall.@zone[-1].masq='1'
uci set firewall.@zone[-1].network='wg0'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='vpn'

uci commit firewall
/etc/init.d/firewall restart
/etc/init.d/network restart
```

### Step 4 - Verify Tunnel
```bash
wg show
```

You should see a **peer** with a recent **handshake**. If not, double-check your private key (must be base64, not hex) and server IP.

---

## Phase 6: Policy Routing (Route LAN Traffic Through VPN)

This is the most critical part. We use **fwmark-based policy routing** to ensure:
- LAN client traffic → marked → routed through `wg0` (VPN) ✅
- Router's own traffic (SSH, DNS) → stays on main table ✅
- WireGuard tunnel packets → never loop back through VPN ✅

### Step 1 - Persistent Route in Table 1
```bash
uci add network route
uci set network.@route[-1].interface='wg0'
uci set network.@route[-1].target='0.0.0.0/0'
uci set network.@route[-1].table='1'
uci commit network
```

### Step 2 - Firewall Mark Rule (mark LAN traffic before routing)
```bash
uci add firewall rule
uci set firewall.@rule[-1].name='mark_lan_vpn'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].target='MARK'
uci set firewall.@rule[-1].set_mark='0x1'
uci commit firewall
/etc/init.d/firewall restart
```

### Step 3 - Hotplug Script (persistent ip rule + nft prerouting mark)
```bash
cat > /etc/hotplug.d/iface/30-vpnroute << 'EOF'
#!/bin/sh
if [ "$ACTION" = "ifup" ] && [ "$INTERFACE" = "wg0" ]; then
    ip rule add fwmark 0x1 lookup 1 priority 100 2>/dev/null
    nft add table inet prerouting_mark 2>/dev/null
    nft add chain inet prerouting_mark prerouting '{ type filter hook prerouting priority mangle; }' 2>/dev/null
    nft add rule inet prerouting_mark prerouting iifname "br-lan" meta mark set 0x1 2>/dev/null
fi
if [ "$ACTION" = "ifdown" ] && [ "$INTERFACE" = "wg0" ]; then
    ip rule del fwmark 0x1 lookup 1 priority 100 2>/dev/null
    nft delete table inet prerouting_mark 2>/dev/null
fi
EOF
chmod +x /etc/hotplug.d/iface/30-vpnroute
```

### Step 4 - Apply Now (without reboot)
```bash
ip rule add fwmark 0x1 lookup 1 priority 100
nft add table inet prerouting_mark
nft add chain inet prerouting_mark prerouting '{ type filter hook prerouting priority mangle; }'
nft add rule inet prerouting_mark prerouting iifname "br-lan" meta mark set 0x1
/etc/init.d/network restart
```

### Step 5 - Verify
```bash
ip rule show
# Should show: 100: from all fwmark 0x1 lookup 1

ip route show table 1
# Should show: default dev wg0 proto static scope link

nft list table inet prerouting_mark
# Should show the prerouting chain with br-lan mark rule
```

Then check from a connected device: [nordvpn.com/what-is-my-ip](https://nordvpn.com/what-is-my-ip/) - should show **"Your internet traffic is secure and protected."**

---

## Phase 7: Rename WiFi SSID

```bash
uci set wireless.@wifi-iface[0].ssid='YourName_VPN_USA'
uci commit wireless
wifi down
wifi up
```

---

## Changing VPN Server

To switch to a different NordVPN server:

```bash
uci set network.@wireguard_wg0[0].endpoint_host='NEW_SERVER_IP'
uci set network.@wireguard_wg0[0].public_key='NEW_PUBLIC_KEY'
uci commit network
/etc/init.d/network restart
```

---

## Troubleshooting

### Lost SSH / No Internet After Routing Changes
The router booted into a broken state. Use **failsafe mode**:

1. Unplug power
2. Plug back in, immediately press reset button rapidly
3. Set PC static IP: `192.168.1.111`, subnet `255.255.255.0`, gateway `192.168.1.1`
4. SSH: `ssh root@192.168.1.1`
5. Run `mount_root`
6. Remove bad config, then `reboot`

### Tunnel Up But No VPN IP
Check if prerouting mark table exists:
```bash
nft list table inet prerouting_mark
```
If missing, re-run the nft commands from Phase 6 Step 4.

### Private Key Rejected
Make sure the key is **base64 encoded** (contains letters, numbers, `+`, `/`, ends with `=`). A hex string (only 0-9 and a-f) will not work.

### Netflix Blocking VPN
Switch to a different US server - some IPs are blocked by Netflix. Get another server IP using the PowerShell command in Phase 4 Step 3 and update the endpoint.

### Speed is Low
- Check the cable between your main router and the Linksys WAN port - must be **Cat5e or Cat6**
- A Cat5 cable caps at 100 Mbps
- Run `ip link show wan` to verify link speed

---

## Known Limitations

- **mwlwifi driver** - The WRT3200ACM uses a proprietary Marvell WiFi chip. The OpenWrt driver has a performance ceiling (~83-180 Mbps WiFi). This is a known unfixable limitation due to the closed-source firmware blob.
- **High latency to US** - Israel → Los Angeles adds ~280ms. Use a European server for lower latency if streaming/gaming performance matters more than a US IP.
