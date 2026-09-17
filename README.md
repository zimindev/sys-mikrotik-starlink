# 🌐 MikroTik RouterOS 7 + Starlink

Universal basic configuration for a MikroTik router running **RouterOS 7** with **Starlink as the Internet connection**.

The configuration provides:

* 🌐 Starlink Internet via `ether1`
* 🔀 LAN bridge
* 📡 Ethernet LAN
* 📶 WLAN / Wi-Fi
* 📡 DHCP Server
* 🌍 DNS
* 🔥 Basic firewall
* 🔄 NAT
* 🛡️ LAN-only management
* 🕒 NTP
* 💾 Configuration backup

> ⚠️ **Important:** Interface names and wireless configuration depend on the MikroTik model and RouterOS version. Check the existing configuration before applying the commands.

---

# 🗺️ Network Topology

```text
                         STARLINK
                            │
                            │ Ethernet
                            ▼
                         ether1
                    ┌──────────────┐
                    │              │
                    │   MikroTik   │
                    │  RouterOS 7  │
                    │              │
                    └──────┬───────┘
                           │
                       bridge-LAN
                           │
              ┌────────────┼────────────┐
              │            │            │
            ether2       ether3       ether4/5
              │            │            │
              └────────────┴────────────┘
                           │
                          LAN
                    192.168.88.0/24
                           │
                ┌──────────┼──────────┐
                │          │          │
               PC       Switch       Wi-Fi
                                      │
                                  SSID: Name-WIFI
```

---

# 🔌 Interface Assignment

| Interface    | Purpose         |
| ------------ | --------------- |
| `ether1`     | 🌐 Starlink WAN |
| `ether2`     | 💻 LAN          |
| `ether3`     | 💻 LAN          |
| `ether4`     | 💻 LAN          |
| `ether5`     | 💻 LAN          |
| `bridge-LAN` | 🔀 LAN bridge   |
| `wifi1`      | 📶 WLAN 2.4 GHz |
| `wifi2`      | 📶 WLAN 5 GHz   |

> Adapt the Ethernet and Wi-Fi interfaces to the actual MikroTik model.

---

# ⚙️ Configuration

## 1. Router Identity

```rsc
/system identity set name="MikroTik-Starlink"
```

---

# 🔀 2. LAN Bridge

If the router already has a LAN bridge, use the existing bridge instead of creating another one.

Example:

```rsc
/interface bridge add name=bridge-LAN
```

Add LAN ports:

```rsc
/interface bridge port add bridge=bridge-LAN interface=ether2
/interface bridge port add bridge=bridge-LAN interface=ether3
/interface bridge port add bridge=bridge-LAN interface=ether4
/interface bridge port add bridge=bridge-LAN interface=ether5
```

Verify:

```rsc
/interface bridge port print
```

---

# 🏠 3. LAN Address

```rsc
/ip address add \
    address=192.168.88.1/24 \
    interface=bridge-LAN \
    comment="LAN Gateway"
```

LAN gateway:

```text
192.168.88.1
```

---

# 🌐 4. Starlink WAN

Connect the Starlink Ethernet cable to `ether1`.

Starlink WAN uses DHCP:

```rsc
/ip dhcp-client add \
    interface=ether1 \
    add-default-route=yes \
    use-peer-dns=no \
    default-route-distance=1 \
    comment="WAN - Starlink"
```

Check:

```rsc
/ip dhcp-client print detail
```

Expected:

```text
status=bound
```

RouterOS DHCP client automatically receives the address and default gateway from the upstream DHCP server.

---

# 📡 5. DHCP Server

Create the LAN address pool:

```rsc
/ip pool add \
    name=LAN-Pool \
    ranges=192.168.88.10-192.168.88.254
```

Create DHCP Server:

```rsc
/ip dhcp-server add \
    name=LAN-DHCP \
    interface=bridge-LAN \
    address-pool=LAN-Pool \
    lease-time=1d \
    disabled=no
```

Configure DHCP network:

```rsc
/ip dhcp-server network add \
    address=192.168.88.0/24 \
    gateway=192.168.88.1 \
    dns-server=192.168.88.1
```

Verify:

```rsc
/ip dhcp-server print
/ip dhcp-server lease print
```

---

# 📶 6. WLAN / Wi-Fi

> ⚠️ This section applies to MikroTik devices using the **RouterOS 7 `/interface wifi`** configuration system. The WiFi menu was introduced in RouterOS 7.13. Compatible 802.11ax devices require the `wifi-qcom` package.

## 6.1 Check Wi-Fi Interfaces

```rsc
/interface wifi print
```

Typical configuration:

```text
wifi1 → 2.4 GHz
wifi2 → 5 GHz
```

Both wireless interfaces should be connected to the LAN bridge.

---

## 6.2 Wi-Fi Security

Create a WPA2/WPA3 security profile:

```rsc
/interface wifi security add \
    name=wifi-security \
    authentication-types=wpa2-psk,wpa3-psk \
    passphrase="CHANGE-THIS-WIFI-PASSWORD"
```

> 🔑 Replace `CHANGE-THIS-WIFI-PASSWORD` with a strong unique Wi-Fi password.

---

## 6.3 Wi-Fi Configuration

Create the Wi-Fi configuration:

```rsc
/interface wifi configuration add \
    name=wifi-LAN \
    mode=ap \
    ssid="Name-WIFI" \
    security=wifi-security \
    datapath.bridge=bridge-LAN
```

---

## 6.4 Enable 2.4 GHz and 5 GHz

```rsc
/interface wifi set wifi1 \
    disabled=no \
    configuration=wifi-LAN

/interface wifi set wifi2 \
    disabled=no \
    configuration=wifi-LAN
```

The result:

```text
             MikroTik
                 │
          ┌──────┴──────┐
          │             │
        wifi1         wifi2
       2.4 GHz        5 GHz
          │             │
          └──────┬──────┘
                 │
             bridge-LAN
                 │
          192.168.88.0/24
```

The official MikroTik WiFi documentation provides the corresponding RouterOS 7 Wi-Fi AP configuration model.

---

## 6.5 Check Connected Wi-Fi Clients

```rsc
/interface wifi registration-table print
```

Detailed information:

```rsc
/interface wifi registration-table print detail
```

Traffic monitoring:

```rsc
/interface monitor-traffic wifi1
/interface monitor-traffic wifi2
```

---

# 🌍 7. DNS

Use Cloudflare and Google DNS:

```rsc
/ip dns set \
    allow-remote-requests=yes \
    servers=1.1.1.1,8.8.8.8
```

Verify:

```rsc
/ip dns print
```

---

# 🔗 8. Interface Lists

Create interface lists:

```rsc
/interface list add name=LAN
/interface list add name=WAN
```

Add LAN:

```rsc
/interface list member add \
    list=LAN \
    interface=bridge-LAN
```

Add Starlink:

```rsc
/interface list member add \
    list=WAN \
    interface=ether1
```

Verify:

```rsc
/interface list member print
```

---

# 🔄 9. NAT

Enable Internet access for LAN and Wi-Fi clients:

```rsc
/ip firewall nat add \
    chain=srcnat \
    action=masquerade \
    out-interface=ether1 \
    comment="NAT - Starlink"
```

`masquerade` is designed for dynamic WAN addresses, such as addresses received through DHCP.

---

# 🔥 10. Firewall

## INPUT

```rsc
/ip firewall filter add \
    chain=input \
    action=accept \
    connection-state=established,related,untracked \
    comment="INPUT - established, related"

/ip firewall filter add \
    chain=input \
    action=drop \
    connection-state=invalid \
    comment="INPUT - drop invalid"

/ip firewall filter add \
    chain=input \
    action=accept \
    in-interface-list=LAN \
    comment="INPUT - allow LAN"

/ip firewall filter add \
    chain=input \
    action=drop \
    in-interface-list=WAN \
    comment="INPUT - block WAN"

/ip firewall filter add \
    chain=input \
    action=drop \
    comment="INPUT - drop everything else"
```

## FORWARD

```rsc
/ip firewall filter add \
    chain=forward \
    action=accept \
    connection-state=established,related,untracked \
    comment="FORWARD - established, related"

/ip firewall filter add \
    chain=forward \
    action=drop \
    connection-state=invalid \
    comment="FORWARD - drop invalid"

/ip firewall filter add \
    chain=forward \
    action=drop \
    in-interface-list=WAN \
    out-interface-list=LAN \
    comment="FORWARD - block WAN to LAN"

/ip firewall filter add \
    chain=forward \
    action=accept \
    in-interface-list=LAN \
    out-interface-list=WAN \
    comment="FORWARD - LAN to Internet"

/ip firewall filter add \
    chain=forward \
    action=drop \
    comment="FORWARD - drop everything else"
```

> ⚠️ Firewall rules are processed from top to bottom. Review existing firewall rules before adding these to an already-configured router. MikroTik documents the firewall as a stateful/stateless filtering system used to control traffic to and through the router.

---

# 🛡️ 11. Management Security

Disable unnecessary services:

```rsc
/ip service set telnet disabled=yes
/ip service set www disabled=yes
/ip service set api disabled=yes
/ip service set api-ssl disabled=yes
```

Allow SSH only from LAN:

```rsc
/ip service set ssh address=192.168.88.0/24
```

Allow WinBox only from LAN:

```rsc
/ip service set winbox address=192.168.88.0/24
```

Verify:

```rsc
/ip service print
```

---

# 🖥️ 12. MAC WinBox & Neighbor Discovery

Allow MAC management only from LAN:

```rsc
/tool mac-server set allowed-interface-list=LAN

/tool mac-server mac-winbox set \
    allowed-interface-list=LAN
```

Restrict Neighbor Discovery:

```rsc
/ip neighbor discovery-settings set \
    discover-interface-list=LAN
```

---

# 🕒 13. Time & NTP

```rsc
/system clock set time-zone-name=Europe/Kyiv

/system ntp client set enabled=yes

/system ntp client servers add \
    address=time.cloudflare.com

/system ntp client servers add \
    address=pool.ntp.org
```

---

# 💾 14. Backup

Create a binary backup:

```rsc
/system backup save name=before-starlink
```

Create a readable configuration export:

```rsc
/export file=before-starlink
```

Verify:

```rsc
/file print
```

---

# 🧪 15. Verification

### Interfaces

```rsc
/interface print
```

### Bridge

```rsc
/interface bridge port print
```

### LAN

```rsc
/ip address print
```

### Starlink

```rsc
/ip dhcp-client print detail
```

### DHCP

```rsc
/ip dhcp-server print
/ip dhcp-server lease print
```

### Wi-Fi

```rsc
/interface wifi print
/interface wifi registration-table print
```

### Routing

```rsc
/ip route print detail
```

### DNS

```rsc
/ip dns print
```

### NAT

```rsc
/ip firewall nat print
```

### Firewall

```rsc
/ip firewall filter print
```

---

# 🌐 16. Internet Test

After connecting Starlink to `ether1`:

```rsc
/ip dhcp-client print detail
```

Expected:

```text
status=bound
```

Test Internet:

```rsc
/ping 1.1.1.1
```

Test DNS:

```rsc
/ping google.com
```

Test from a LAN/Wi-Fi client:

```text
IP address: 192.168.88.x
Gateway:    192.168.88.1
DNS:        192.168.88.1
Internet:   Working
```

---

# ✅ Final Result

After configuration:

```text
                         STARLINK
                            │
                            ▼
                         ether1
                            │
                    ┌──────────────┐
                    │   MikroTik   │
                    │  RouterOS 7  │
                    └──────┬───────┘
                           │
                       bridge-LAN
                           │
              ┌────────────┼────────────┐
              │            │            │
            Ethernet     Ethernet      Wi-Fi
              │            │            │
              └────────────┴────────────┘
                           │
                    192.168.88.0/24
```

### Network Parameters

| Parameter          | Value                          |
| ------------------ | ------------------------------ |
| 🌐 WAN             | Starlink                       |
| 🔌 WAN interface   | `ether1`                       |
| 📡 WAN protocol    | DHCP                           |
| 🏠 LAN             | `192.168.88.0/24`              |
| 🚪 Gateway         | `192.168.88.1`                 |
| 📡 DHCP pool       | `192.168.88.10-192.168.88.254` |
| 🌍 DNS             | `1.1.1.1`, `8.8.8.8`           |
| 🔄 NAT             | Masquerade                     |
| 📶 Wi-Fi           | 2.4 GHz + 5 GHz                |
| 📡 SSID            | `Name-WIFI`                        |
| 🔥 Firewall        | Enabled                        |
| 🛡️ WAN management | Blocked                        |
| 🖥️ LAN management | Allowed                        |

---

# ⚠️ Security Notes

* 🔒 Do not expose WinBox or SSH directly to the Internet.
* 🔑 Use a strong unique administrator password.
* 🔐 Use a strong unique Wi-Fi password.
* 🚫 Do not publish passwords or Wi-Fi keys in GitHub.
* 🛡️ Keep WAN management disabled.
* 💾 Keep a known-good backup before major configuration changes.
* 🔍 Review existing configuration before adding duplicate bridges, DHCP servers, IP addresses, interface lists or firewall rules.

---

# 📚 Official Documentation

* [MikroTik RouterOS Documentation](https://manual.mikrotik.com/docs/?utm_source=chatgpt.com)
* [MikroTik Getting Started](https://help.mikrotik.com/docs/spaces/ROS/pages/328119/Getting+started?utm_source=chatgpt.com)
* [MikroTik WiFi Documentation](https://help.mikrotik.com/docs/spaces/ROS/pages/224559120/WiFi?utm_source=chatgpt.com)
* [MikroTik Wireless Documentation](https://help.mikrotik.com/docs/spaces/ROS/pages/1409138/Wireless?utm_source=chatgpt.com)
* [MikroTik DHCP Documentation](https://help.mikrotik.com/docs/spaces/ROS/pages/24805500/DHCP?utm_source=chatgpt.com)
* [MikroTik Firewall Documentation](https://help.mikrotik.com/docs/spaces/ROS/pages/250708066/Firewall?utm_source=chatgpt.com)
* [MikroTik NAT Documentation](https://help.mikrotik.com/docs/spaces/ROS/pages/3211299/NAT?utm_source=chatgpt.com)
* [MikroTik Hardware Manuals](https://help.mikrotik.com/docs/spaces/UM/overview?utm_source=chatgpt.com)
* [Starlink Support](https://www.starlink.com/support?utm_source=chatgpt.com)

---

## 🚀 Ready

**Connect Starlink → `ether1` → MikroTik receives WAN via DHCP → LAN and Wi-Fi receive addresses via DHCP → Internet works.**
