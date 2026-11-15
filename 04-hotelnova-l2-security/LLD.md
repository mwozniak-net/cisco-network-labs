# Lab 04 – HotelNova – Low-Level Design

## 1. Device inventory

| Hostname | Model | Role | Management IP |
|---|---|---|---|
| R1-GW | ISR 2911 | gateway, inter-VLAN routing, DHCPv4, SLAAC | 10.94.99.1 |
| D1-SW | Catalyst 2960-24TT | distribution, root VLAN 10/99 | 10.94.99.2 |
| D2-SW | Catalyst 2960-24TT | distribution, root VLAN 20 | 10.94.99.3 |
| A1-SW | Catalyst 2960-24TT | access, L2 security | 10.94.99.4 |
| PC-ADMIN | PC-PT | administrator | 10.94.99.50 |
| PC-STAFF | PC-PT | staff | DHCPv4 + SLAAC, VLAN 10 |
| PC-GUEST | PC-PT | guest | DHCPv4 + SLAAC, VLAN 20 |
| ROGUE-DHCP | ISR 2911 | attacker: DHCP server 192.168.66.0/24 | 192.168.66.1 |

## 2. Interfaces

| Device | Interface | Type | Function / addressing |
|---|---|---|---|
| R1-GW | Gi0/0 | physical | trunk to D1 (no IP) |
| R1-GW | Gi0/0.10 | subinterface | 10.94.10.1/24, 2001:DB8:94:10::1/64 |
| R1-GW | Gi0/0.20 | subinterface | 10.94.20.1/24, 2001:DB8:94:20::1/64 |
| R1-GW | Gi0/0.99 | subinterface, native | 10.94.99.1/24 |
| D1-SW | Port-channel1 | port-channel | LACP to D2, trunk |
| D1-SW | Port-channel2 | port-channel | PAgP to A1, trunk |
| D1-SW | Fa0/23–24 | physical, Po1 members | `channel-group 1 mode active` |
| D1-SW | Fa0/21–22 | physical, Po2 members | `channel-group 2 mode desirable` |
| D1-SW | Gi0/1 | physical | trunk to R1-GW |
| D1-SW | Fa0/1 | physical | access 99, PC-ADMIN |
| D2-SW | Port-channel1 | port-channel | LACP to D1, trunk |
| D2-SW | Fa0/23–24 | physical, Po1 members | `channel-group 1 mode passive` |
| D2-SW | Fa0/20 | physical | backup trunk to A1 |
| A1-SW | Port-channel2 | port-channel | PAgP to D1, trunk |
| A1-SW | Fa0/21–22 | physical, Po2 members | `channel-group 2 mode desirable` |
| A1-SW | Fa0/24 | physical | backup trunk to D2 |
| A1-SW | Fa0/1, Fa0/2, Fa0/10 | physical, access | PC-STAFF (10), PC-GUEST (20), ROGUE-DHCP (20) |
| D1 / D2 / A1 | Vlan99 | SVI | .2 / .3 / .4, gateway 10.94.99.1 |

All trunks and port-channels: `switchport mode trunk`, native VLAN 99, allowed 10,20,99, `switchport nonegotiate`. Unused ports: access 999, `description UNUSED`, `shutdown`.

## 3. VLANs (VTP transparent, all switches)

| VLAN | Name |
|---:|---|
| 10 | STAFF |
| 20 | GUEST |
| 99 | MGMT |
| 999 | PARKING |

## 4. EtherChannel

| Channel | Protocol | Mode (side A / side B) | Members | Link |
|---|---|---|---|---|
| Po1 | LACP | active (D1) / passive (D2) | Fa0/23, Fa0/24 | D1–D2 |
| Po2 | PAgP | desirable (D1) / desirable (A1) | Fa0/21, Fa0/22 | D1–A1 |

## 5. Spanning tree (Rapid PVST+)

| VLAN | Root | D1 priority | D2 priority | A1 root port | A1 blocked port |
|---:|---|---:|---:|---|---|
| 10 | D1 | 24576 | 28672 | Po2 (cost 12) | Fa0/24 |
| 20 | D2 | 28672 | 24576 | Fa0/24 (cost 19) | Po2 |
| 99 | D1 | 24576 | 28672 | Po2 | Fa0/24 |

## 6. Addressing services (R1-GW)

| Pool | Network | Excluded | Gateway | DNS |
|---|---|---|---|---|
| STAFF_DHCP | 10.94.10.0/24 | .1–.20 | 10.94.10.1 | 8.8.8.8 |
| GUEST_DHCP | 10.94.20.0/24 | .1–.20 | 10.94.20.1 | 8.8.8.8 |

IPv6: `ipv6 unicast-routing`. The subinterfaces send Router Advertisements, so hosts build their address with SLAAC (EUI-64), e.g. PC-GUEST 2001:DB8:94:20:201:42FF:FE7B:6DB6.

## 7. Access-layer security (A1-SW)

```
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option
ip arp inspection vlan 20                    ! extension (DAI)
interface range fa0/21 - 22 , fa0/24
 ip dhcp snooping trust
 ip arp inspection trust
interface range fa0/1 - 2 , fa0/10
 ip dhcp snooping limit rate 10
 spanning-tree portfast
 spanning-tree bpduguard enable
interface fa0/1
 switchport port-security
 switchport port-security mac-address sticky
 ! maximum 1 and violation shutdown are the IOS defaults (show port-security: Max 1, Shutdown)
```

### Target IOS configuration for features Packet Tracer does not support

```
! DHCP snooping / DAI trust on the logical bundle (PT accepts it only on members)
interface Port-channel2
 ip dhcp snooping trust
 ip arp inspection trust
! IP Source Guard on the PC-GUEST port, based on snooping bindings
interface fa0/2
 ip verify source
! Brute-force protection on the switches
login block-for 60 attempts 3 within 60
```

### Restoring an err-disabled port (port security, BPDU Guard)

Find the cause with `show interfaces status err-disabled` and `show port-security interface fa0/1`, and remove the foreign device. Then:

```
interface fa0/1
 shutdown
 no shutdown
```

Alternatively, `errdisable recovery cause psecure-violation` with an interval.

## 8. Management and hardening (all devices)

- **SSH:** `ip domain-name hotelnova.local`, RSA keys, `ip ssh version 2`.
- **Accounts:** `username netadmin privilege 15 secret …`, `enable secret`, `service password-encryption`.
- **Remote access:** `access-list 1 permit 10.94.99.0 0.0.0.255`. `line vty 0 15` uses `access-class 1 in`, `login local`, `transport input ssh` and `exec-timeout 5 0`.
- **Console:** `exec-timeout 5 0`.
- **R1-GW:** `login block-for 10 attempts 3 within 60`. The ACL `sl_def_acl` is generated automatically by IOS for quiet mode.

## 9. Packet Tracer notes

- **Snooping trust on the bundle:** trust works only on the member ports. DHCP replies arriving on Po2 are dropped, so PC-STAFF (VLAN 10 over Po2) gets an address only when Po2 is down (test T7).
- **2960 and `privilege 15`:** the switch does not display `privilege 15` in running-config. `show privilege` after an SSH login returns 15.
- **Rejected commands:** `ip verify source` and, on the 2960, `login block-for` are rejected.
