# Lab 01 – BookNook – Low-Level Design

## 1. Device inventory

| Hostname | Model | Role | Management IP |
|---|---|---|---|
| R1-BOOK | ISR4331 (IOS XE 16.6) | gateway, inter-VLAN routing, DHCP | 10.90.99.1 |
| S1-BOOK | Catalyst 2960-24TT | central switch | 10.90.99.11 (VLAN 99) |
| S2-BOOK | Catalyst 2960-24TT | access switch | 10.90.99.12 (VLAN 99) |
| Server | Server-PT | DNS, HTTP | 10.90.99.10 |
| PC_ADMIN | PC-PT | administration | 10.90.99.69 (static) |
| PC_STAFF | PC-PT | staff | DHCP (VLAN 10) |
| PC_CLIENT | PC-PT | customer terminal | DHCP (VLAN 20) |

## 2. Physical connections

| Device A | Port | Device B | Port | Type |
|---|---|---|---|---|
| R1-BOOK | Gi0/0/0 | S1-BOOK | Gi0/1 | trunk |
| S1-BOOK | Fa0/24 | S2-BOOK | Fa0/24 | trunk |
| S1-BOOK | Fa0/1 | PC_ADMIN | Fa0 | access VLAN 99 |
| S1-BOOK | Fa0/2 | Server | Fa0 | access VLAN 99 |
| S2-BOOK | Fa0/1 | PC_STAFF | Fa0 | access VLAN 10 |
| S2-BOOK | Fa0/2 | PC_CLIENT | Fa0 | access VLAN 20 |

## 3. VLANs and trunks

| VLAN | Name | Present on |
|---:|---|---|
| 10 | STAFF | S1, S2 |
| 20 | CLIENTS | S1, S2 |
| 99 | MGMT | S1, S2 |
| 999 | PARKING | S1, S2 |

Trunks (R1–S1 Gi0/1, S1–S2 Fa0/24): `switchport mode trunk`, `switchport nonegotiate`, allowed VLANs 10,20,99, native VLAN 99. On R1 the subinterface Gi0/0/0.99 uses `encapsulation dot1Q 99 native`.

## 4. IP addressing

| Device | Interface | IPv4 | IPv6 |
|---|---|---|---|
| R1-BOOK | Gi0/0/0.10 | 10.90.10.1/24 | 2001:DB8:90:10::1/64 |
| R1-BOOK | Gi0/0/0.20 | 10.90.20.1/24 | 2001:DB8:90:20::1/64 |
| R1-BOOK | Gi0/0/0.99 | 10.90.99.1/24 | 2001:DB8:90:99::1/64 |
| S1-BOOK | Vlan99 | 10.90.99.11/24, gw .1 | – |
| S2-BOOK | Vlan99 | 10.90.99.12/24, gw .1 | – |
| Server | Fa0 | 10.90.99.10/24, gw .1 | – |
| PC_ADMIN | Fa0 | 10.90.99.69/24, gw .1 | – |

`ipv6 unicast-routing` is enabled on R1.

## 5. DHCP (R1-BOOK)

| Pool | Network | Default router | DNS | Excluded |
|---|---|---|---|---|
| POOL_STAFF | 10.90.10.0/24 | 10.90.10.1 | 10.90.99.10 | .1–.10 |
| POOL_CLIENTS | 10.90.20.0/24 | 10.90.20.1 | 10.90.99.10 | .1–.10 |

## 6. Switch port configuration

| Switch | Ports | Mode / VLAN | Protection |
|---|---|---|---|
| S1-BOOK | Fa0/1–2 | access 99 | PortFast, BPDU Guard |
| S1-BOOK | Fa0/3–23, Gi0/2 | access 999, shutdown | – |
| S2-BOOK | Fa0/1 | access 10 | PortFast, BPDU Guard |
| S2-BOOK | Fa0/2 | access 20 | port security: sticky MAC, max 1, violation restrict; PortFast, BPDU Guard |
| S2-BOOK | Fa0/3–23, Gi0/1–2 | access 999, shutdown | – |

## 7. Services (Server)

- **DNS:** A record `intranet.booknook.local` → 10.90.99.10.
- **HTTP/HTTPS:** enabled (intranet page).
- **Other services:** DHCP on the server is disabled; addressing comes from R1.

## 8. Management and hardening (all network devices)

- **Domain and keys:** `ip domain-name booknook.local`, RSA keys generated, `ip ssh version 2`.
- **Accounts:** local user `netadmin` (secret) and `enable secret`.
- **Remote access:** `access-list 1 permit 10.90.99.0 0.0.0.255`. All vty lines use `access-class 1 in`, `login local` and `transport input ssh`.
- **Passwords:** `service password-encryption` is on.
- **Banner:** MOTD reads "authorised access only".

## 9. Verification commands

```
R1-BOOK# show ip interface brief
R1-BOOK# show ip dhcp binding
S1-BOOK# show interfaces trunk
S2-BOOK# show port-security interface Fa0/2
PC> ipconfig /renew
PC> nslookup intranet.booknook.local
PC_ADMIN> ssh -l netadmin 10.90.99.12
```

Results after the fixes are listed in [README.md](README.md#verification-packet-tracer-90-after-fixes).
