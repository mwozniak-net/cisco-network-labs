# Lab 02 – AgroTrans – Low-Level Design

## 1. Device inventory

| Hostname | Model | Role | Management IP |
|---|---|---|---|
| R1-HQ | ISR 2911 | HQ router, OSPF, ISP edge | 10.91.10.1 |
| R2-WH | ISR 2901 | warehouse router | 10.91.20.1 |
| R3-ISP | ISR 2901 | simulated ISP (Lo0 = 8.8.8.8) | – |
| S1-HQ | Catalyst 2960-24TT | HQ access switch | 10.91.10.2 (VLAN 10) |
| S2-WH | Catalyst 2960-24TT | warehouse access switch | 10.91.20.2 (VLAN 20) |
| PC-HQ | PC-PT | HQ host | 10.91.10.50 |
| PC-WH | PC-PT | warehouse host | 10.91.20.50 |

## 2. Physical connections

| Device A | Port | Device B | Port | Link |
|---|---|---|---|---|
| R1-HQ | Gi0/0 | S1-HQ | Fa0/24 | HQ LAN |
| R1-HQ | Gi0/1 | R2-WH | Gi0/1 | LINE_A |
| R1-HQ | Se0/3/0 (DCE, clock rate 64000) | R2-WH | Se0/3/0 | LINE_B |
| R1-HQ | Gi0/2 | R3-ISP | Gi0/0 | ISP |
| R2-WH | Gi0/0 | S2-WH | Fa0/24 | WH LAN |
| S1-HQ | Fa0/1 | PC-HQ | Fa0 | access VLAN 10 |
| S2-WH | Fa0/1 | PC-WH | Fa0 | access VLAN 20 |

## 3. IP addressing

| Device | Interface | IPv4 | IPv6 |
|---|---|---|---|
| R1-HQ | Gi0/0 | 10.91.10.1/24 | 2001:DB8:91:10::1/64 |
| R1-HQ | Gi0/1 | 10.91.0.1/30 | 2001:DB8:91:1::1/64 |
| R1-HQ | Se0/3/0 | 10.91.0.5/30 | 2001:DB8:91:2::1/64 |
| R1-HQ | Gi0/2 | 198.51.100.2/30 | 2001:DB8:91:3::2/64 |
| R2-WH | Gi0/0 | 10.91.20.1/24 | 2001:DB8:91:20::1/64 |
| R2-WH | Gi0/1 | 10.91.0.2/30 | 2001:DB8:91:1::2/64 |
| R2-WH | Se0/3/0 | 10.91.0.6/30 | 2001:DB8:91:2::2/64 |
| R3-ISP | Gi0/0 | 198.51.100.1/30 | 2001:DB8:91:3::1/64 |
| R3-ISP | Lo0 | 8.8.8.8/32 | 2001:DB8:FF::8/128 |
| S1-HQ | Vlan10 | 10.91.10.2/24, gw .1 | – |
| S2-WH | Vlan20 | 10.91.20.2/24, gw .1 | – |

## 4. Routing

### OSPFv2 (process 1, area 0)

| Router | Router ID | Networks | Non-passive | Other |
|---|---|---|---|---|
| R1-HQ | 1.1.1.1 | 10.91.10.0/24, 10.91.0.0/30 | Gi0/1 | `default-information originate` |
| R2-WH | 2.2.2.2 | 10.91.20.0/24, 10.91.0.0/30 | Gi0/1 | – |

### Static routes

| Router | Prefix | Next hop | AD | Purpose |
|---|---|---|---:|---|
| R1-HQ | 0.0.0.0/0 | 198.51.100.1 | 1 | internet |
| R1-HQ | 10.91.20.0/24 | 10.91.0.6 | 150 | backup via LINE_B |
| R2-WH | 10.91.10.0/24 | 10.91.0.5 | 150 | backup via LINE_B |
| R2-WH | 0.0.0.0/0 | 10.91.0.5 | 150 | backup default (primary is O*E2 from R1) |
| R1-HQ | ::/0 | 2001:DB8:91:3::1 | 1 | internet |
| R1-HQ | 2001:DB8:91:20::/64 | Gi0/1 2001:DB8:91:1::2 | 1 | primary |
| R1-HQ | 2001:DB8:91:20::/64 | Se0/3/0 2001:DB8:91:2::2 | 150 | backup |
| R2-WH | 2001:DB8:91:10::/64, ::/0 | Gi0/1 2001:DB8:91:1::1 | 1 | primary |
| R2-WH | 2001:DB8:91:10::/64, ::/0 | Se0/3/0 2001:DB8:91:2::1 | 150 | backup |
| R3-ISP | 10.91.0.0/16 | 198.51.100.2 | 1 | return route to the customer (worksheet task 3) |
| R3-ISP | 2001:DB8:91::/48 | 2001:DB8:91:3::2 | 1 | return route, customer IPv6 block |

There is no NAT. Hosts reach 8.8.8.8 with their own addresses, and R3 routes the replies back through its return routes.

## 5. Switches

| Switch | VLANs | Ports |
|---|---|---|
| S1-HQ | 10 HQ_LAN, 999 UNUSED | Fa0/1 access 10 PC-HQ (PortFast, BPDU Guard), Fa0/24 access 10 uplink to R1, Fa0/2–23 and Gi0/1–2 access 999, UNUSED, shutdown |
| S2-WH | 20 WH_LAN, 999 PARKING | Fa0/1 access 20 PC-WH (PortFast, BPDU Guard), Fa0/24 access 20 uplink to R2, rest access 999, UNUSED, shutdown |

## 6. Management and hardening

- **SSH:** `ip domain-name agrotrans.local`, RSA keys generated, `ip ssh version 2`.
- **Accounts:** local user `netadmin` (privilege 15, secret) and `enable secret` on R1, R2, S1 and S2. `service password-encryption` is on.
- **Remote access:** `access-list 1 permit 10.91.10.0 0.0.0.255`. vty 0–4 and 5–15 use `access-class 1 in`, `login local` and `transport input ssh`.
- **ISP-facing interface:** `no cdp enable` on R1-HQ Gi0/2. CDP stays enabled inside the network.

## 7. Verification commands

```
R1-HQ# show ip ospf neighbor
R1-HQ# show ip protocols                       ! passive interfaces
R2-WH# show ip route                           ! default route O*E2 from R1
R1-HQ# show ip route 10.91.20.0
R1-HQ(config)# interface Gi0/1 / shutdown      ! failover test
R2-WH# ping 2001:DB8:91:10::1
R1-HQ# show cdp neighbors                      ! R3-ISP not listed
PC-WH> ping 8.8.8.8 / ping 2001:DB8:FF::8
```

## 8. Packet Tracer notes

- **Account privilege on the 2960:** the switch accepts `privilege 15` in `username netadmin privilege 15 secret …`, but its running-config does not show it.
- **IPv6 static routes:** a route with only a next-hop address is not withdrawn when the exit interface goes down. Fully specified routes (interface + next hop) fix this.

Results after the fixes are listed in [README.md](README.md#verification-packet-tracer-90-after-fixes).
