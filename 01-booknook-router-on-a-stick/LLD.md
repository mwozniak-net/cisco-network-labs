# Lab 01 – BookNook – Low-Level Design

## 1. Device inventory

| Hostname | Model | Role | Management IP |
|---|---|---|---|
| R1-BOOK | ISR4331 (IOS XE 16.6) | gateway, inter-VLAN routing, DHCP | 10.90.99.1 |
| S1-BOOK | Catalyst 2960-24TT | central switch | 10.90.99.2 |
| S2-BOOK | Catalyst 2960-24TT | access switch | 10.90.99.3 |
| SRV-INFO | Server-PT | DNS, HTTP | 10.90.99.10 |
| PC-ADMIN | PC-PT | administration | 10.90.99.50 |
| PC-STAFF | PC-PT | staff | DHCP / 2001:DB8:90:10::10 |
| PC-CLIENT | PC-PT | customer terminal | DHCP / 2001:DB8:90:20::10 |

## 2. Cabling

| Device A | Port | Device B | Port | Mode |
|---|---|---|---|---|
| R1-BOOK | Gi0/0/0 | S1-BOOK | Gi0/1 | trunk |
| S1-BOOK | Fa0/24 | S2-BOOK | Fa0/24 | trunk |
| S1-BOOK | Fa0/1 | PC-ADMIN | Fa0 | access 99 |
| S1-BOOK | Fa0/2 | SRV-INFO | Fa0 | access 99 |
| S2-BOOK | Fa0/1 | PC-STAFF | Fa0 | access 10 |
| S2-BOOK | Fa0/2 | PC-CLIENT | Fa0 | access 20 |

The worksheet names R1's port Gi0/0. On the ISR4331 used here it is Gi0/0/0.

## 3. Interfaces and addressing

| Device | Interface | Type | IPv4 | IPv6 |
|---|---|---|---|---|
| R1-BOOK | Gi0/0/0 | physical, trunk to S1 | – | – |
| R1-BOOK | Gi0/0/0.10 | subinterface, dot1Q 10 | 10.90.10.1/24 | 2001:DB8:90:10::1/64 |
| R1-BOOK | Gi0/0/0.20 | subinterface, dot1Q 20 | 10.90.20.1/24 | 2001:DB8:90:20::1/64 |
| R1-BOOK | Gi0/0/0.99 | subinterface, dot1Q 99 native | 10.90.99.1/24 | 2001:DB8:90:99::1/64 |
| R1-BOOK | Gi0/0/1, Gi0/0/2 | unused | – (shutdown) | – |
| S1-BOOK | Vlan99 | SVI | 10.90.99.2/24, gw 10.90.99.1 | – |
| S2-BOOK | Vlan99 | SVI | 10.90.99.3/24, gw 10.90.99.1 | – |

## 4. VLANs and switch ports

| VLAN | Name | S1-BOOK ports | S2-BOOK ports |
|---:|---|---|---|
| 10 | STAFF | – | Fa0/1 |
| 20 | CLIENTS | – | Fa0/2 |
| 99 | MGMT | Fa0/1, Fa0/2 | – |
| 999 | PARKING | Fa0/3–23, Gi0/2 (shutdown, UNUSED) | Fa0/3–23, Gi0/1–2 (shutdown, UNUSED) |

Trunks: `switchport mode trunk`, `switchport trunk native vlan 99`, `switchport trunk allowed vlan 10,20,99`, `switchport nonegotiate`.

| Port | Protection |
|---|---|
| S1 Fa0/1, Fa0/2 | PortFast, BPDU Guard |
| S2 Fa0/1 (PC-STAFF) | PortFast, BPDU Guard |
| S2 Fa0/2 (PC-CLIENT) | PortFast, BPDU Guard, port security: max 1, sticky, violation restrict |

Restoring a port after BPDU Guard puts it in err-disabled: `shutdown` then `no shutdown` on the interface, after the cause is removed.

## 5. DHCPv4 (R1-BOOK)

```
ip dhcp excluded-address 10.90.10.1 10.90.10.10
ip dhcp excluded-address 10.90.20.1 10.90.20.10
ip dhcp pool POOL_STAFF
 network 10.90.10.0 255.255.255.0
 default-router 10.90.10.1
 dns-server 10.90.99.10
ip dhcp pool POOL_CLIENTS
 network 10.90.20.0 255.255.255.0
 default-router 10.90.20.1
 dns-server 10.90.99.10
```

## 6. Services (SRV-INFO)

- **DNS:** A record `intranet.booknook.local` → 10.90.99.10.
- **HTTP:** enabled.
- **DHCP on the server:** disabled.

## 7. Management and hardening (R1, S1, S2)

```
ip domain-name booknook.local
crypto key generate rsa            ! modulus min. 1024 (worksheet requirement)
ip ssh version 2
username netadmin privilege 15 secret <password>
enable secret <password>
service password-encryption
access-list 1 permit 10.90.99.0 0.0.0.255
line vty 0 15                      ! 0 4 on the ISR4331
 access-class 1 in
 login local
 transport input ssh
banner motd #Dostep tylko dla uprawnionych - <hostname>#
```

## 8. Packet Tracer notes

- **Account privilege on the 2960:** the switch accepts `username netadmin privilege 15 secret …` but shows it as `username netadmin secret 5 …` in running-config. After an SSH login, `show privilege` returns 15 (test T7).
- **Interface names:** the ISR4331 uses Gi0/0/x names instead of the worksheet's Gi0/x.
