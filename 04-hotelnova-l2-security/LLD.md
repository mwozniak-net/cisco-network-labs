# Lab 04 – HotelNova – Low-Level Design

## 1. Device inventory

| Hostname | Model | Role | Management IP |
|---|---|---|---|
| R1-GW | ISR 2911 | gateway, inter-VLAN routing, DHCP | 10.94.99.1 |
| D1-SW | Catalyst 2960-24TT | distribution, STP root VLAN 10/99 | 10.94.99.2 |
| D2-SW | Catalyst 2960-24TT | distribution, STP root VLAN 20 | 10.94.99.3 |
| A1-SW | Catalyst 2960-24TT | access, L2 security | 10.94.99.4 |
| PC-ADMIN | PC-PT | administrator | 10.94.99.50 (static) |
| PC-STAFF | PC-PT | staff | DHCP, VLAN 10 |
| PC-GUEST | PC-PT | guest | DHCP, VLAN 20 |
| ROUGE-ROUTER | ISR 2911 | attacker (rogue DHCP 192.168.1.0/24) | – |

## 2. Physical connections

| Device A | Port | Device B | Port | Function |
|---|---|---|---|---|
| R1-GW | Gi0/0 | D1-SW | Gi0/1 | trunk 10,20,99 |
| D1-SW | Fa0/23–24 | D2-SW | Fa0/23–24 | Po1, LACP (D1 active, D2 passive) |
| D1-SW | Fa0/21–22 | A1-SW | Fa0/21–22 | Po2, PAgP (desirable / desirable) |
| A1-SW | Fa0/24 | D2-SW | Fa0/20 | backup trunk |
| D1-SW | Fa0/1 | PC-ADMIN | Fa0 | access 99 |
| A1-SW | Fa0/1 | PC-STAFF | Fa0 | access 10 |
| A1-SW | Fa0/2 | PC-GUEST | Fa0 | access 20 |
| A1-SW | Fa0/10 | ROUGE-ROUTER | Gi0/0 | access 20 (attack port) |

All trunks: `switchport mode trunk`, `switchport nonegotiate`, allowed VLANs 10,20,99, native VLAN 99.

## 3. VLANs and addressing

| VLAN | Name | IPv4 | IPv6 (R1-GW) | Gateway |
|---:|---|---|---|---|
| 10 | STAFF | 10.94.10.0/24 | 2001:DB8:94:10::/64 | Gi0/0.10 – .1 |
| 20 | GUEST | 10.94.20.0/24 | 2001:DB8:94:20::/64 | Gi0/0.20 – .1 |
| 99 | MGMT (native) | 10.94.99.0/24 | – | Gi0/0.99 – .1 |
| 999 | PARKING | – | – | – |

### DHCP (R1-GW)

| Pool | Network | Excluded | Gateway | DNS | Domain |
|---|---|---|---|---|---|
| STAFF_DHCP | 10.94.10.0/24 | .1–.20 | 10.94.10.1 | 8.8.8.8 | hotelnova.local |
| GUEST_DHCP | 10.94.20.0/24 | .1–.20 | 10.94.20.1 | 8.8.8.8 | hotelnova.local |

## 4. Spanning tree (Rapid PVST+)

| Switch | VLAN 10 | VLAN 20 | VLAN 99 |
|---|---:|---:|---:|
| D1-SW | 24576 (root) | 28672 | 24576 (root) |
| D2-SW | 28672 | 24576 (root) | 28672 |
| A1-SW | 32768 | 32768 | 32768 |

A1-SW Po2 has `spanning-tree vlan 10 cost 100`. This is a Packet Tracer workaround, described in the README. Resulting forwarding on A1: VLAN 10 and 20 use Fa0/24, VLAN 99 uses Po2.

## 5. Access switch A1-SW

| Feature | Configuration |
|---|---|
| DHCP snooping | `ip dhcp snooping`, `ip dhcp snooping vlan 10,20`, `no ip dhcp snooping information option` |
| Trusted ports | Fa0/21, Fa0/22 (Po2 members), Fa0/24 – `ip dhcp snooping trust`, `ip arp inspection trust` |
| Rate limit | `ip dhcp snooping limit rate 10` on Fa0/1, Fa0/2, Fa0/10 |
| DAI | `ip arp inspection vlan 20` |
| Port security | Fa0/1: `switchport port-security`, sticky MAC, max 1 (default), violation shutdown (default) |
| Edge ports | Fa0/1, Fa0/2, Fa0/10: PortFast, BPDU Guard |
| Unused | Fa0/3–9, Fa0/11–20, Fa0/23, Gi0/1–2: access 999, shutdown |

## 6. Management and hardening (all devices)

- **SSH and accounts:** `ip domain-name hotelnova.local`, RSA keys generated, `ip ssh version 2`, local user `netadmin` (secret) and `enable secret`.
- **Remote access:** `access-list 1 permit 10.94.99.0 0.0.0.255`. vty 0–15 use `access-class 1 in`, `login local`, `transport input ssh` and `exec-timeout 5 0`.
- **Console:** `exec-timeout 5 0`.
- **R1-GW:** `login block-for 10 attempts 3 within 60`. The quiet-mode ACL `sl_def_acl` is created automatically by IOS.
- **Other:** `service password-encryption`. Banner: "Dostep tylko dla uprawnionych" (authorised access only).

## 7. Attack scenario

ROUGE-ROUTER is connected to A1-SW Fa0/10 (guest VLAN) and runs a DHCP pool for 192.168.1.0/24 with itself as the default gateway. Without DHCP snooping, a guest could receive an address from the attacker and send all its traffic through it (man-in-the-middle). With snooping, Fa0/10 is untrusted, so DHCPOFFER/ACK messages from it are dropped and clients receive addresses only from R1-GW.

## 8. Verification commands

```
D1-SW# show etherchannel summary
D1-SW# show spanning-tree vlan 10
A1-SW# show ip dhcp snooping
A1-SW# show ip dhcp snooping binding
A1-SW# show port-security interface Fa0/1
A1-SW# show interfaces trunk
PC-GUEST> ipconfig /renew
```

Results after the fixes are listed in [README.md](README.md#verification-packet-tracer-90-after-fixes).
