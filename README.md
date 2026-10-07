# Cisco Routing Labs

Network labs I built and verified in GNS3. Each lab has the topology, the router configuration at every stage, the `show` output, and packet captures.

| Lab | Topics | Status |
|---|---|---|
| [nat-dhcp](nat-dhcp/) | Static NAT, PAT, dynamic NAT, pool exhaustion, DHCP server | Done |
| [ospf](ospf/) | Single-area OSPF, convergence time after link failure, Hello/Dead tuning, multi-area, virtual link | Done |
| [rip](rip/) | RIPv1 and RIPv2, hop count against link speed, convergence compared with OSPF, split horizon, routing loop, count-to-infinity | Done |
| vlan-intervlan | VLANs, 802.1Q trunk, router-on-a-stick | Planned |

Tools: GNS3, Cisco IOS 15.2 (7200), VPCS, Wireshark.

Cisco IOS images are not included in this repository.
