# Lab 03 – QuickBite – Low-Level Design

## 1. Device inventory

| Hostname | Model | Role | Address |
|---|---|---|---|
| R1-EDGE | ISR 2911 | gateway, NAT, ACLs, DHCP | 10.92.99.1 (mgmt) |
| S1-LAN | Catalyst 2960-24TT | access switch | 10.92.99.2 (VLAN 99) |
| R2-ISP | ISR 2911 | simulated ISP | 192.0.2.1 |
| SRV-WEB | Server-PT | DMZ web server (HTTP/HTTPS) | 172.16.92.10 |
| SRV-INET | Server-PT | "internet" server: DNS, HTTP | 8.8.8.8 |
| PC-STAFF | PC-PT | staff | DHCP, VLAN 10 |
| PC-GUEST | PC-PT | guest | DHCP, VLAN 30 |
| PC-ADMIN | PC-PT | administrator | 10.92.99.50 |
| PC-OUTSIDE | PC-PT | internet user | 198.51.100.50 |

## 2. Physical connections

| Device A | Port | Device B | Port | Notes |
|---|---|---|---|---|
| R1-EDGE | Gi0/0 | S1-LAN | Gi0/1 | trunk 10,30,99, native 99 |
| R1-EDGE | Gi0/1 | R2-ISP | Gi0/0 | WAN /30 |
| R1-EDGE | Gi0/2 | SRV-WEB | Fa0 | DMZ |
| S1-LAN | Fa0/1 | PC-STAFF | Fa0 | access 10 |
| S1-LAN | Fa0/2 | PC-GUEST | Fa0 | access 30 |
| S1-LAN | Fa0/3 | PC-ADMIN | Fa0 | access 99 |
| R2-ISP | Gi0/1 | SRV-INET | Fa0 | 8.8.8.0/24 |
| R2-ISP | Gi0/2 | PC-OUTSIDE | Fa0 | 198.51.100.0/24 |

## 3. VLANs and addressing

| VLAN | Name | Subnet | Gateway (R1 subinterface) |
|---:|---|---|---|
| 10 | STAFF | 10.92.10.0/24 | Gi0/0.10 – 10.92.10.1 |
| 30 | GUEST | 10.92.30.0/24 | Gi0/0.30 – 10.92.30.1 |
| 99 | MGMT (native) | 10.92.99.0/24 | Gi0/0.99 – 10.92.99.1 |
| 999 | PARKING | – | – |
| – | DMZ | 172.16.92.0/24 | Gi0/2 – 172.16.92.1 |
| – | WAN | 192.0.2.0/30 | Gi0/1 – 192.0.2.2 |

DHCP on R1-EDGE: pools `VLAN10` and `VLAN30`. Each pool hands out the .1 gateway and DNS 8.8.8.8, and addresses .1–.10 are excluded.

## 4. NAT

| Type | Rule |
|---|---|
| PAT | `ip nat inside source list NAT-PAT interface Gi0/1 overload` – NAT-PAT permits 10.92.10.0/24 and 10.92.30.0/24 |
| Static (port forwarding) | tcp 172.16.92.10:80 → 192.0.2.2:80 |
| Static (port forwarding) | tcp 172.16.92.10:443 → 192.0.2.2:443 |
| Inside | Gi0/0.10, Gi0/0.30, Gi0/0.99, Gi0/2 |
| Outside | Gi0/1 |

Routing: `ip route 0.0.0.0 0.0.0.0 192.0.2.1`.

## 5. Access control lists

### ACL_GUEST – inbound on Gi0/0.30

| Seq | Rule |
|---:|---|
| 1 | permit udp any eq bootpc any eq bootps (DHCP) |
| 2 | permit udp 10.92.30.0/24 host 8.8.8.8 eq domain |
| 3 | deny ip any 10.92.10.0/24 |
| 4 | deny ip any 10.92.99.0/24 |
| 5 | deny ip any 172.16.92.0/24 |
| 6 | permit tcp 10.92.30.0/24 any eq www |
| 7 | permit tcp 10.92.30.0/24 any eq 443 |
| 8 | deny ip any any |

### DMZ-OUT – outbound on Gi0/2 (traffic entering the DMZ)

| Seq | Rule |
|---:|---|
| 1 | permit tcp any host 172.16.92.10 eq www |
| 2 | permit tcp any host 172.16.92.10 eq 443 |
| 3 | permit tcp 10.92.99.0/24 host 172.16.92.10 eq 22 |
| 4 | deny ip any any (target IOS: `deny ip any any log`, which Packet Tracer rejects) |

Server-PT has no SSH server. The SSH rule is part of the design, and its effect can only be shown on the HTTP counters.

### ACL_SSH – vty 0–15

`permit 10.92.99.0 0.0.0.255`, with `login local` and `transport input ssh` on the vty lines.

## 6. Switch ports (S1-LAN)

| Ports | Mode / VLAN | Protection |
|---|---|---|
| Fa0/1 | access 10 | port security sticky, violation restrict; PortFast, BPDU Guard |
| Fa0/2 | access 30 | PortFast, BPDU Guard |
| Fa0/3 | access 99 | PortFast, BPDU Guard |
| Fa0/4–24, Gi0/2 | access 999, shutdown | – |
| Gi0/1 | trunk 10,30,99, native 99, nonegotiate | – |

## 7. Servers

- **SRV-WEB:** HTTP and HTTPS enabled, gateway 172.16.92.1.
- **SRV-INET:** DNS enabled with the A record `www.quickbite.com` → 192.0.2.2. HTTP is enabled, gateway 8.8.8.1.

## 8. Management and logging

- **SSH and accounts:** `ip domain-name quickbite`, RSA keys, `ip ssh version 2`, `netadmin` privilege 15, `enable secret`, `service password-encryption`.
- **Logging:** `logging buffered`. Packet Tracer accepts it but does not display it in running-config.
- **Packet Tracer:** the 2960 accepts `privilege 15` for the local account but does not display it in running-config.

## 9. Verification commands

```
R1-EDGE# show ip nat translations
R1-EDGE# show access-lists
R1-EDGE# show ip dhcp binding
PC-GUEST> ipconfig /renew
PC-GUEST> nslookup www.quickbite.com
PC-GUEST> ping 172.16.92.10        ! expected: unreachable (ACL)
PC-OUTSIDE> browser http://192.0.2.2
```

Results after the fixes are listed in [README.md](README.md#verification-packet-tracer-90-after-fixes).
