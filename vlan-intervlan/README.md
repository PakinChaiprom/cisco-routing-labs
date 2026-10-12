English | [ภาษาไทย](README.th.md)

# VLANs, 802.1Q Trunking and Inter-VLAN Routing (GNS3)

Two switches, seven hosts and one router. The lab starts with one flat network, splits it into VLANs, carries the VLANs between switches on an 802.1Q trunk, breaks the trunk on purpose with a native VLAN mismatch, and then routes between the VLANs with a router-on-a-stick and with switch virtual interfaces (SVIs). Every step is checked with `show` output and Wireshark.

> Individual lab, built in GNS3 (Oct 2026). Switches: Cisco 3745 with the NM-16ESW 16-port switch module, IOS 12.4(25d). Router: Cisco 7200, IOS 15.2. Hosts: VPCS.

## Topology

![Topology](topology.png)

| VLAN | Name | Hosts | Parts 1–2 (one subnet for all) | Part 3 (one subnet per VLAN) |
|---|---|---|---|---|
| 10 | SALES | PC1, PC2 (SW1), PC5 (SW2) | 192.168.1.11, .12, .15 /24 | 192.168.10.0/24, gateway .1 |
| 20 | ENG | PC3, PC4 (SW1), PC6 (SW2) | 192.168.1.13, .14, .16 /24 | 192.168.20.0/24, gateway .1 |
| 30 | MGMT | PC7 (SW2) | – | 192.168.30.0/24, gateway .1 |
| 99 | NATIVE | none | native VLAN of every trunk from part 2e | native VLAN of every trunk |

| Link | Mode |
|---|---|
| PC1–PC4 – SW1 Fa1/1 to Fa1/4, PC5–PC7 – SW2 Fa1/1 to Fa1/3 | access |
| SW1 Fa1/14 – SW2 Fa1/14 | trunk, 802.1Q, native 99 |
| SW1 Fa1/0 – R1 Fa0/0 | trunk, 802.1Q, native 99 |

In parts 1 and 2 every host is on 192.168.1.0/24 on purpose. If two hosts with addresses in the same subnet cannot reach each other, the thing that separates them is the VLAN, not IP.

## Results at a glance

| Part | Test | Result |
|---|---|---|
| 1 | VLAN 10 and 20 on one switch | PC1 → PC3 fails although both are in 192.168.1.0/24. PC1's ARP broadcast reached PC3's port once before the VLANs, and zero times after. |
| 2a | Two links between the switches, both in VLAN 1 | STP blocks SW2 Fa1/15. Path cost and sender bridge ID were equal; the sender's port ID decided. |
| 2c | One 802.1Q trunk | The same echo request is 102 bytes on the trunk and 98 bytes on the access link: the 4-byte tag. |
| 2e | Native VLAN 10 on one side, 20 on the other | No traffic leaked. PVST+ detected the mismatch and blocked VLAN 10 and 20 on the trunk. |
| 3a | Router-on-a-stick | All VLANs reach each other. One ping between VLANs puts 4 frames on the router link, each packet crossing it twice with a different tag. |
| 3b | SVIs on SW1 | Same reachability, and no ICMP on the router link at all. |

## 1. VLANs on one switch

```
SW1# vlan database
SW1(vlan)# vlan 10 name SALES
SW1(vlan)# vlan 20 name ENG
SW1(vlan)# exit
APPLY completed.

interface range FastEthernet1/1 - 2
 switchport mode access
 switchport access vlan 10
interface range FastEthernet1/3 - 4
 switchport mode access
 switchport access vlan 20
```

| | Before VLANs | After VLANs |
|---|---|---|
| PC1 → PC2 (both VLAN 10) | replies | replies |
| PC1 → PC3 (VLAN 10 → 20) | replies | `host (192.168.1.13) not reachable` |
| PC3 → PC4 (both VLAN 20) | – | replies |
| PC1's ARP for PC2, seen on PC3's link | 1 frame | 0 frames (192 s capture) |

**Why `host not reachable` and not a timeout.** PC1 and PC3 are in the same subnet, so PC1 sends an ARP broadcast for PC3's MAC address. The switch floods that broadcast only to ports in VLAN 10, so it never reaches PC3, nobody answers, and PC1 never sends the ICMP packet at all. A timeout would mean the ICMP packet went out and no reply came back. Part 3 shows that case.

The capture on PC3's port also shows an STP BPDU every 2 seconds and a CDP frame every 60 seconds, even though only a PC is connected there. After the port moved to VLAN 20, the Bridge ID inside the BPDU changed from `32768 / c4:01:07:b4:00:00` to `32768 / c4:01:07:b4:00:02`: the switch runs one STP instance per VLAN. This switch module tells the instances apart by using a different MAC address for each one, not by adding the VLAN number to the priority like newer Catalyst switches (32768 + 20 = 32788).

## 2. VLANs across two switches

### 2a. Two links, both in VLAN 1: STP blocks one

Two links between the same two switches in the same VLAN form a loop.

- **Root bridge:** SW1, because the priorities are equal (32768) and SW1's MAC address `c401.07b4.0002` is lower than SW2's `c402.1ab6.0000`
- **SW2 Fa1/14:** root port, forwarding
- **SW2 Fa1/15:** blocking

SW2 picks its root port by comparing the two links step by step:

| Step | Compared | Fa1/14 | Fa1/15 |
|---|---|---|---|
| 1 | Path cost to the root | 19 | 19 |
| 2 | Bridge ID of the sender | SW1 | SW1 |
| 3 | **Port ID of the sender** (SW1's port) | **128.55** | 128.56 |

Step 3 decides it. The value comes from `Designated port id is 128.55` in SW2's `show spanning-tree`. It is the port ID on SW1's side, not SW2's own port.

### 2b. One access link per VLAN

With Fa1/14 in VLAN 10 and Fa1/15 in VLAN 20 on both switches, PC1 reaches PC5, PC3 reaches PC6, PC1 cannot reach PC6, and nothing is blocked because each VLAN has only one path. It works, but every new VLAN would need another cable and another pair of ports.

### 2c. One 802.1Q trunk

```
interface FastEthernet1/14
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

![802.1Q tag in Wireshark](vlan_tag_wireshark.png)

| Field | Value |
|---|---|
| Ethernet Type | `0x8100` (802.1Q) |
| Priority | 0 (Best Effort) |
| DEI | 0 |
| VLAN ID | 10 for PC1 ↔ PC5, 20 for PC3 ↔ PC6 |
| Type after the tag | `0x0800` (IPv4) |
| Frame length of the same echo request | **102 bytes on the trunk, 98 bytes on PC1's access link** |

Control traffic on the trunk (native VLAN still 1):

| Frames | Destination | Tag | Bridge ID |
|---|---|---|---|
| STP for VLAN 1, IEEE format | `01:80:c2:00:00:00` | none | `c4:01:07:b4:00:02` |
| STP for VLAN 1, Cisco PVST+ format | `01:00:0c:cc:cc:cd` | none | `c4:01:07:b4:00:02` |
| STP for VLAN 10 | `01:00:0c:cc:cc:cd` | 10 | `c4:01:07:b4:00:00` |
| STP for VLAN 20 | `01:00:0c:cc:cc:cd` | 20 | `c4:01:07:b4:00:01` |
| CDP | `01:00:0c:cc:cc:cc` | none | – |

Each VLAN's STP instance sends its own BPDUs, tagged so that the other switch knows which instance they belong to. Only SW1's BPDUs appear: SW1 is the root in every VLAN, and SW2's trunk port is a root port, which receives BPDUs but does not send them. No DTP frames appeared; `show interfaces Fa1/14 switchport` reports `Negotiation of Trunking: Disabled`.

### 2d. Allowed VLANs

```
SW1(config-if)# switchport trunk allowed vlan 1,10
Command rejected: Bad VLAN allowed list. You have to include all default vlans, e.g. 1-2,1002-1005.
SW1(config-if)# switchport trunk allowed vlan 1,10,1002-1005
```

The switch module insists on VLAN 1 and 1002–1005 in the list. With VLAN 20 removed from the trunk, PC3 → PC6 failed and `show interfaces trunk` listed only VLAN 1 and 10 as active. `switchport trunk allowed vlan 1-1005` restored it.

### 2e. Native VLAN mismatch

I set the native VLAN to 10 on SW1's side of the trunk and to 20 on SW2's side, to see whether untagged VLAN 10 frames from SW1 would land in VLAN 20 on SW2.

They did not. The switches logged the mismatch as soon as the native VLAN changed:

```
%SPANTREE-2-RECV_PVID_ERR: Received BPDU with inconsistent peer vlan id 1 on FastEthernet1/14 VLAN10.
%SPANTREE-2-BLOCK_PVID_PEER: Blocking FastEthernet1/14 on VLAN1. Inconsistent peer vlan.
%SPANTREE-2-BLOCK_PVID_LOCAL: Blocking FastEthernet1/14 on VLAN10. Inconsistent local vlan.
```

```
Port 55 (FastEthernet1/14) of VLAN10 is broken (Port VLAN ID Mismatch)
Port 55 (FastEthernet1/14) of VLAN20 is broken (Port VLAN ID Mismatch)
```

- Only VLAN 1 stayed in forwarding state on the trunk
- PC1 could reach neither PC5 nor PC6
- The 22-minute capture of the trunk contains no untagged ARP or ICMP frame

**How the switch knows.** A Cisco PVST+ BPDU carries the VLAN it belongs to. SW1 sent its VLAN 10 BPDU untagged, because VLAN 10 was its native VLAN. SW2 put the untagged frame into its own native VLAN 20, read VLAN 10 inside the BPDU, and blocked the port for that VLAN. A switch without this check would have forwarded PC1's frames into VLAN 20.

**Fix:** VLAN 99 `NATIVE` on both switches and `switchport trunk native vlan 99` on both ends. No host is in VLAN 99, so an untagged frame that ends up on the trunk reaches nobody. After the change, the VLAN 1 PVST+ BPDU is tagged with 1 and the VLAN 99 BPDU is the untagged one.

## 3. Routing between VLANs

Each VLAN now gets its own subnet. VLAN 30 has to exist on SW1 as well, even with no host there, or the trunk to the router does not carry it.

### 3a. Router-on-a-stick

```
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
interface FastEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
interface FastEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
interface FastEthernet0/0.99
 encapsulation dot1Q 99 native
```

`encapsulation dot1Q 10` ties the subinterface to VLAN 10: it accepts frames tagged 10 and tags what it sends with 10. The three subnets appear in `show ip route` as connected routes; no routing protocol is needed.

| PC1 to | Result | TTL | Trace |
|---|---|---|---|
| 192.168.10.1 (gateway) | replies | 255 | – |
| PC2, same VLAN | replies | 64 | 1 hop |
| PC3 and PC6, VLAN 20 | replies | 63 | 2 hops: 192.168.10.1, then the host |
| PC7, VLAN 30 on SW2 | replies | 63 | – |
| 192.168.20.17, no such host | `timeout` | – | – |

**One echo request, seen twice on the router link:**

| | Frame 309, into R1 | Frame 312, out of R1 |
|---|---|---|
| VLAN | 10 | 20 |
| Source MAC | PC1 `00:50:79:66:68:00` | R1 `ca:03:0a:0e:00:00` |
| Destination MAC | R1 `ca:03:0a:0e:00:00` | PC3 `00:50:79:66:68:02` |
| TTL | 64 | 63 |

The IP addresses stay the same. The router replaces the Ethernet header and the tag and lowers the TTL. One ping between VLANs (request and reply) produces 4 frames on the router link, and a ping inside VLAN 10 produces none. All inter-VLAN traffic shares this one link, twice: the weak point of a router-on-a-stick.

**`timeout` here, `host not reachable` in part 1.** I mistyped 192.168.20.17 for PC7's address. That host does not exist, but the result was a timeout, not `host not reachable`:

1. The destination is in another subnet, so PC1 ARPs for its gateway, gets an answer, and sends the ICMP packet to R1
2. R1 then ARPs for 192.168.20.17 in VLAN 20; the capture shows 6 requests from 192.168.20.1 and no answer
3. R1 drops the packet without telling PC1
4. PC1 has already sent its packet and waits in vain

`host not reachable` means the host's own ARP failed, so the problem is in its own subnet. A timeout means the packet left, and the problem is further along the path.

### 3b. Routing on the switch (SVIs)

With R1's Fa0/0 shut down:

```
ip routing
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
interface Vlan20
 ip address 192.168.20.1 255.255.255.0
interface Vlan30
 ip address 192.168.30.1 255.255.255.0
```

The gateway addresses stay the same, so nothing changes on the PCs. Pings between VLANs still work with TTL 63 and a 2-hop trace, and the capture on the router link now contains no ICMP at all, only flooded ARP and BPDUs. SW1 routes between its own VLANs. Response times were similar to the router-on-a-stick, because both are emulated routers in GNS3. The capture shows where the traffic goes, not how fast real hardware would be.

## Notes on the emulated switch

GNS3 has no Catalyst switch image for Dynamips, so the switches are Cisco 3745 routers with the NM-16ESW switch module. It behaves like a Catalyst switch for this lab, with these differences:

| Task | Catalyst | NM-16ESW |
|---|---|---|
| Create a VLAN | `vlan 10` in global config | `vlan database`, then `vlan 10 name SALES`, then `exit` |
| Show VLANs | `show vlan brief` | `show vlan-switch brief` |
| Allowed VLANs on a trunk | any list | must include 1 and 1002–1005 |
| STP Bridge ID per VLAN | priority + VLAN number | separate MAC address per VLAN, assigned in the order the instances start |

Limitations of the emulation that I ran into:

- VLANs are kept in `vlan.dat`, not in the running-config, and are lost when the GNS3 project is closed. The port configuration survives, so the ports sit in VLANs that no longer exist. Pasting [`configs/vlan_database.txt`](configs/vlan_database.txt) after every start brings everything back.
- `show mac-address-table` listed only VLAN 1 entries (for example R1's MAC address on Fa1/0), never the hosts in VLAN 10, 20 or 30, even though forwarding worked.
- Deleting a link in GNS3 did not bring the switch port down. `show spanning-tree` still listed Fa1/15 as forwarding after its cable was gone, so I shut the port down instead.

## Problems I ran into

| Problem | Cause and fix |
|---|---|
| `end` rejected inside `vlan database` | `exit` leaves that mode and applies the changes. |
| `switchport trunk allowed vlan 1,10` rejected | The list must include VLAN 1 and 1002–1005. |
| Every VLAN gone after reopening the project | `vlan.dat` is not kept by GNS3. Paste the `vlan database` commands again. |
| Switch port still forwarding after deleting its link in GNS3 | The emulated port does not go down. `shutdown` on both ends instead. |

## Repository layout

```
vlan-intervlan/
├── README.md
├── topology.png
├── vlan_tag_wireshark.png
├── vlan-intervlan.gns3
├── configs/
│   ├── vlan_database.txt         VLANs to paste after every start
│   ├── 1_one_switch/             SW1 (part 1)
│   ├── 2_trunk_native99/         SW1, SW2 (end of part 2)
│   └── 3_router_on_a_stick/      R1, SW1, SW2 (part 3a)
├── outputs/
│   ├── 00_base.txt
│   ├── 01_no_vlan.txt
│   ├── 02_vlan_one_switch.txt
│   ├── 03_two_links_vlan1.txt
│   ├── 04_access_link_per_vlan.txt
│   ├── 05_trunk_allowed_vlan.txt
│   ├── 06_native_vlan_mismatch.txt
│   ├── 07_router_on_a_stick.txt
│   └── 08_svi.txt
└── pcaps/
    ├── 01_no_vlan_pc3_link.pcapng
    ├── 02_vlan_pc3_link.pcapng
    ├── 05_trunk.pcapng
    ├── 05_access_pc1_link.pcapng
    ├── 06_native_mismatch_trunk.pcapng
    ├── 07_router_on_a_stick_r1_link.pcapng
    └── 08_svi_r1_link.pcapng
```

The output files are copied from the console. Typing mistakes and rejected commands are left in.
