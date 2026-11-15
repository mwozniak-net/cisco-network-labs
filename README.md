# Cisco network labs (Packet Tracer)

Four Cisco Packet Tracer labs from my networking practice in autumn 2026. Each lab includes the working `.pkt` file, the final device configurations, a High-Level Design and a Low-Level Design.

After building the labs I reviewed them again, fixed the errors I found and re-tested every fix in Packet Tracer. Each lab's README lists what was wrong, what it broke, how it was fixed and the test results.

| Lab | Scenario | Main topics | Key fix found in review |
|---|---|---|---|
| [01 – BookNook](01-booknook-router-on-a-stick) | bookstore, 3 VLANs | router-on-a-stick, DHCP, DNS, SSH + ACL, port security | management SVI in the wrong subnet; wrong DNS server in a DHCP pool |
| [02 – AgroTrans](02-agrotrans-wan-failover) | HQ + warehouse, 2 WAN links | OSPF, floating static routes (IPv4/IPv6), PAT, serial link | IPv6 had no working failover; missing NAT |
| [03 – QuickBite](03-quickbite-nat-dmz-acl) | restaurant with a public web server | PAT, static NAT, DMZ, extended ACLs, guest isolation | guest ACL blocked DHCP |
| [04 – HotelNova](04-hotelnova-l2-security) | hotel, rogue device on a guest port | DHCP snooping, DAI, port security, EtherChannel (LACP/PAgP), Rapid PVST+ | DHCP snooping vs. EtherChannel in Packet Tracer |

## Repository layout

```
NN-name/
  README.md     scenario, topology, review findings, verification results
  HLD.md        high-level design: requirements, segments, design decisions, limitations
  LLD.md        low-level design: devices, cabling, addressing, routing, ACLs, port maps
  configs/      final running-config of every router and switch
  labNN-*.pkt   Packet Tracer file (opens in Packet Tracer 9.0)
```

## Notes

- **Requirements:** the original task sheets are not included, so the requirements in the HLDs were reconstructed from the configurations.
- **Credentials:** the passwords in the labs are simple lab values used only inside the simulation, and the `.pkt` files have had the CLI command history removed.
- **Packet Tracer limitations:** where Packet Tracer behaves differently from real IOS, the README of the lab says so and gives the configuration for real hardware. Two examples: DHCP snooping trust on a port-channel, and `login quiet-mode access-class`.

---
Miłosz Woźniak · [LinkedIn](https://www.linkedin.com/in/milosz-wozniak-/) · [GitHub](https://github.com/mwozniak-net)
