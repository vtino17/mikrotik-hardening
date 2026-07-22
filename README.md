# MikroTik Hardening

[![CI](https://img.shields.io/github/actions/workflow/status/vtino17/mikrotik-hardening/ci.yml?style=flat-square&label=CI)](https://github.com/vtino17/mikrotik-hardening/actions)

[![License](https://img.shields.io/badge/License-MIT-22AA55?style=flat-square)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/vtino17/mikrotik-hardening?style=flat-square&logo=github)](https://github.com/vtino17/mikrotik-hardening/stargazers)
[![RouterOS](https://img.shields.io/badge/RouterOS-7.x-00AEEF?style=flat-square&logo=mikrotik&logoColor=white)](https://mikrotik.com/)
[![Last commit](https://img.shields.io/github/last-commit/vtino17/mikrotik-hardening?style=flat-square)](https://github.com/vtino17/mikrotik-hardening/commits)
[![Contributions](https://img.shields.io/badge/contributions-welcome-brightgreen?style=flat-square)](CONTRIBUTING.md)

Production-grade security configurations for MikroTik RouterOS v7+.

This guide compiles battle-tested configurations from real deployments — not theory. Each section addresses a specific attack surface and includes copy-paste-ready RouterOS commands.

## Index

1. [Initial Security](#1-initial-security)
2. [Firewall Rulesets](#2-firewall-rulesets)
3. [VLAN Segmentation](#3-vlan-segmentation)
4. [VPN Tunnels](#4-vpn-tunnels)
5. [DNS & Web Filtering](#5-dns--web-filtering)
6. [Bandwidth Management](#6-bandwidth-management)
7. [Logging & Monitoring](#7-logging--monitoring)
8. [Wazuh SIEM Integration](#8-wazuh-siem-integration)
9. [Backup & Disaster Recovery](#9-backup--disaster-recovery)
10. [Audit Checklist](#10-audit-checklist)

---

## 1. Initial Security

### Remove default user & secure admin access

```bash
/user remove admin
/user add name=admin group=full disabled=yes
/user add name=neteng group=full password="<complex-password>"

# SSH hardening
/ip ssh set strong-crypto=yes host-key-size=4096 allow-none-crypto=no
/ip ssh set always-allow-password-login=no

# Disable unused services
/ip service disable telnet,ftp,www,api,api-ssl,winbox
/ip service set ssh port=2222
/ip service set www-ssl port=443
/tool bandwidth-server set enabled=no
/tool mac-server set allowed-interface-list=none
/tool mac-winbox-server set allowed-interface-list=none
```

### NTP & Time sync

```bash
/system ntp client set enabled=yes primary=162.159.200.1 secondary=162.159.200.123
/system clock set time-zone-autodetect=no time-zone-name=Asia/Jakarta
```

---

## 2. Firewall Rulesets

### Base protection (connection tracking + default drop)

```bash
/ip firewall filter
# Default drops
add chain=input connection-state=invalid action=drop comment="Drop invalid"
add chain=input connection-state=established action=accept
add chain=input connection-state=related action=accept
add chain=input protocol=icmp action=accept comment="Allow ICMP"
add chain=input action=drop comment="Drop all else"

# Layer-7 DDoS protection
/ip firewall layer7-protocol
add name=dns-flood regexp="^.+(suspicious\.domain\.com).*$"

# Port scan detection
/ip firewall filter
add chain=input protocol=tcp psd=21,3s,3,1 action=add-src-to-address-list \
    address-list=port-scanners address-list-timeout=1d comment="PSD scan detect"
add chain=input protocol=tcp connection-limit=10,32 action=add-src-to-address-list \
    address-list=port-scanners address-list-timeout=1d comment="Connection limit"
add chain=input src-address-list=port-scanners action=drop comment="Drop scanners"
```

### Forward chain / VLAN isolation

```bash
/ip firewall filter
add chain=forward in-interface=br-management out-interface=br-guest action=drop
add chain=forward in-interface=br-guest out-interface=br-management action=drop
add chain=forward in-interface=br-guest out-interface=br-server action=drop
add chain=forward in-interface=br-iot out-interface=!br-iot action=drop
```

### NAT / Masquerade

```bash
/ip firewall nat
add chain=srcnat out-interface=pppoe-out action=masquerade comment="Internet access"
add chain=dstnat in-interface=pppoe-out protocol=tcp dst-port=443 \
    action=dst-nat to-addresses=192.168.1.10 to-ports=443 comment="Wazuh dashboard"
```

---

## 3. VLAN Segmentation

```bash
# Create VLAN interfaces
/interface vlan add name=vlan-management vlan-id=10 interface=bridge-local
/interface vlan add name=vlan-server vlan-id=20 interface=bridge-local
/interface vlan add name=vlan-guest vlan-id=30 interface=bridge-local
/interface vlan add name=vlan-iot vlan-id=40 interface=bridge-local
/interface vlan add name=vlan-voip vlan-id=50 interface=bridge-local

# Bridge with VLAN filtering
/interface bridge add name=bridge-local vlan-filtering=yes
/interface bridge port add bridge=bridge-local interface=ether2
/interface bridge vlan add bridge=bridge-local tagged=bridge-local untagged=ether2 vlan-ids=10,20,30,40,50

# DHCP per VLAN
/ip dhcp-server add name=dhcp-management interface=vlan-management address-pool=pool-management
/ip pool add name=pool-management ranges=192.168.10.2-192.168.10.254
```

| VLAN | ID | Subnet | Purpose | Internet | Inter-VLAN |
|------|----|--------|---------|----------|------------|
| Management | 10 | 192.168.10.0/24 | Admin devices | ✅ | All |
| Server | 20 | 192.168.20.0/24 | Wazuh, Nginx, DB | ✅ | Mgmt only |
| Guest | 30 | 192.168.30.0/24 | Visitors | ✅ | None |
| IoT | 40 | 192.168.40.0/24 | Smart devices | ✅ | None |
| VoIP | 50 | 192.168.50.0/24 | Phones | ✅ | Mgmt only |

---

## 4. VPN Tunnels

### WireGuard site-to-site

```bash
/interface wireguard add name=wg-office listen-port=51820 private-key="..."
/ip address add address=10.0.1.1/30 interface=wg-office
/interface wireguard peers add interface=wg-office public-key="..." \
    endpoint-address=203.0.113.10 endpoint-port=51820 allowed-address=10.0.2.0/24
/ip route add dst-address=10.0.2.0/24 gateway=wg-office
```

### L2TP/IPsec for remote users

```bash
/interface l2tp-server server set enabled=yes use-ipsec=yes \
    ipsec-secret="<secret>" default-profile=l2tp-profile
/ppp profile add name=l2tp-profile local-address=192.168.100.1 \
    remote-address=192.168.100.2-192.168.100.50 use-encryption=yes
/ppp secret add name=remote-user password="<complex>" service=l2tp profile=l2tp-profile
```

---

## 5. DNS & Web Filtering

### Local DNS cache with blocklist

```bash
/ip dns set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes cache-size=4096

# Block malicious domains
/ip dns static add name=malware.example.com address=0.0.0.0 type=A
/ip dns static add name=phishing.example.com address=0.0.0.0 type=A
```

### AdGuard Home / PiHole upstream

```bash
# Point clients to AdGuard VM at 192.168.20.10
/ip dhcp-server network set 0 dns-server=192.168.20.10

# Fallback to RouterOS DNS if AdGuard is down
/ip dns set servers=192.168.20.10,1.1.1.1
```

---

## 6. Bandwidth Management

### Simple Queue per VLAN

```bash
/queue simple add name=guest-limit target=vlan-guest/30 max-limit=10M/10M \
    burst-limit=20M/20M burst-threshold=8M/8M burst-time=30s

/queue simple add name=iot-limit target=vlan-iot/40 max-limit=5M/5M \
    burst-limit=10M/10M burst-threshold=4M/4M burst-time=30s

/queue simple add name=voip-priority target=vlan-voip/50 max-limit=50M/50M \
    parent=none queue=wireless-default/wireless-default priority=1
```

### PCQ for fair distribution

```bash
/queue type add name=pcq-download kind=pcq pcq-classifier=dst-address
/queue type add name=pcq-upload kind=pcq pcq-classifier=src-address

/queue simple add name=pcq-all target=bridge-local/24 max-limit=100M/100M \
    queue=pcq-upload/pcq-download
```

---

## 7. Logging & Monitoring

### Syslog to remote SIEM

```bash
/system logging action add name=remote-siem target=remote \
    remote=192.168.20.5:514 remote-log-level=info
/system logging add action=remote-siem topics=info,error,warning,firewall
/system logging add action=remote-siem topics=account,critical
```

### SNMP for network monitoring

```bash
/snmp community set public addresses=192.168.20.0/24
/snmp set enabled=yes contact="neteng@example.com" location="DC1"
```

### Email alerts for critical events

```bash
/tool e-mail set address=smtp.example.com port=587 \
    user="neteng@example.com" password="<pw>" start-tls=yes

/system script add name=alert-admin source={
    :log warning "Router rebooted"
    /tool e-mail send to="admin@example.com" \
        subject="[MikroTik] Router rebooted" body="Uptime: [/system resource get uptime]"
}
```

---

## 8. Wazuh SIEM Integration

### RouterOS syslog configuration

```
# On Wazuh agent: /var/ossec/etc/ossec.conf
<localfile>
  <log_format>syslog</log_format>
  <location>192.168.20.5:514</location>
</localfile>
```

### Custom decoder reference

See [wazuh-custom-decoders](https://github.com/vtino17/wazuh-custom-decoders) repo for the complete MikroTik decoder ruleset.

---

## 9. Backup & Disaster Recovery

### Scheduled backup to remote

```bash
/system script add name=backup-cloud source={
    /system backup save name=($ystem.identity..."-"...[/system clock get date])
    /tool fetch url="sftp://192.168.20.5/backups/" \
        user=backup password="<pw>" \
        src-path=($ystem.identity..."-"...[/system clock get date]...".backup")
    :delay 5s
    /file remove [find name~$ystem.identity]
}

/system scheduler add name=sched-backup interval=1d start-time=03:00 \
    on-event=backup-cloud
```

### Export config

```bash
/export file=router-config-20260721
```

---

## 10. Audit Checklist

- [ ] Default admin removed and complex passwords set
- [ ] Unused services disabled (telnet, ftp, api, winbox)
- [ ] SSH on non-standard port, key-only or strong password
- [ ] Connection tracking enabled with stateful rules
- [ ] Port scan detection active
- [ ] VLAN filtering enabled with inter-VLAN rules
- [ ] DDoS protection (connection limits, layer-7)
- [ ] DNS pointing to filtered resolvers
- [ ] Logging to remote SIEM (Wazuh)
- [ ] Firewall rules logged for critical chains
- [ ] NTP configured correctly
- [ ] SNMP restricted to management network
- [ ] Automatic backups scheduled
- [ ] VPN with strong ciphers (WireGuard or IPsec)
- [ ] Firmware updated to latest RouterOS stable
- [ ] Bandwidth management enabled with at least basic queues

---

## References

- [MikroTik Security Manual](https://wiki.mikrotik.com/wiki/Manual:Security)
- [RouterOS v7 Firewall](https://wiki.mikrotik.com/wiki/Manual:IP/Firewall/Filter)
- [CIS MikroTik Benchmark](https://www.cisecurity.org/benchmark/mikrotik)

---

**Maintained by [vtino17](https://github.com/vtino17)** — contributions welcome via PR.

