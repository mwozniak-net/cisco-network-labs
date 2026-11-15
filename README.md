# Cisco network labs (Packet Tracer)

Four practice worksheets I completed in Cisco Packet Tracer while preparing for the **SkillsPoland 2026 national final, Skill 39 – IT Network Systems Administration**. Each worksheet has a time limit (120–180 min), 100 points, and requires an HLD, an LLD and a test protocol. Each lab here contains:

- the working `.pkt` file and the final configuration of every router and switch,
- the worksheet tasks, a review of my saved configuration against them, and the fixes,
- a test protocol with expected and actual results, including negative tests,
- an HLD and an LLD,
- Packet Tracer limitations, with the target IOS configuration.

| Lab | Worksheet | Main topics | Time limit |
|---|---|---|---|
| [01 – BookNook](01-booknook-router-on-a-stick) | 8 | VLANs, trunks, router-on-a-stick, DHCPv4, DNS/HTTP, SSH + VTY ACL, port security | 120 min |
| [02 – AgroTrans](02-agrotrans-wan-failover) | 9 | OSPFv2, default route propagation, floating static routes, serial link, IPv6 static routing | 150 min |
| [03 – QuickBite](03-quickbite-nat-dmz-acl) | 10 | PAT, static PAT, DMZ, named extended ACLs, guest isolation, logging | 165 min |
| [04 – HotelNova](04-hotelnova-l2-security) | 12 | EtherChannel LACP/PAgP, Rapid PVST+, DHCP snooping vs. rogue DHCP, SLAAC, VTP transparent | 180 min |

## Repository layout

```
NN-name/
  README.md     worksheet tasks, review findings and fixes, PT limitations, test protocol
  HLD.md        purpose, requirements, logical design, justified design decisions
  LLD.md        devices, interfaces, addressing, VLANs/STP/EtherChannel/ACL tables, config excerpts
  configs/      final running-config of every router and switch
  labNN-*.pkt   Packet Tracer file (opens in Packet Tracer 9.0)
```

## Notes

- **Credentials:** the passwords are simple values used only inside the simulation, and the CLI command history has been removed from the `.pkt` files.
- **Tests not re-run:** tests that need extra devices connected (port security violation, BPDU Guard) are marked "not re-run".
- **Packet Tracer limitations found while testing:** each lab's README and LLD describes them and gives the target IOS configuration. They include:
  - DHCP snooping trust on a port-channel,
  - IP Source Guard,
  - `deny … log` in ACLs,
  - next-hop-only IPv6 static routes not being withdrawn after a link failure,
  - `privilege 15` not shown on the 2960.

---
Miłosz Woźniak · [LinkedIn](https://www.linkedin.com/in/milosz-wozniak-/) · [GitHub](https://github.com/mwozniak-net)
