[English](README.md) | ภาษาไทย

# Cisco Routing Labs

Lab ด้าน network ที่ผมสร้างและทดสอบเองใน GNS3 แต่ละ lab มี topology, config ของอุปกรณ์ทุกขั้นตอน, ผลคำสั่ง `show` และไฟล์ packet capture

| Lab | หัวข้อ | สถานะ |
|---|---|---|
| [nat-dhcp](nat-dhcp/README.th.md) | Static NAT, PAT, dynamic NAT, pool หมด, DHCP server | เสร็จแล้ว |
| [ospf](ospf/README.th.md) | OSPF แบบ single area, เวลา convergence หลัง link เสีย, ปรับ Hello/Dead, multi-area, virtual link | เสร็จแล้ว |
| [rip](rip/README.th.md) | RIPv1 และ RIPv2, hop count เทียบกับความเร็ว link, convergence เทียบกับ OSPF, split horizon, routing loop, count-to-infinity | เสร็จแล้ว |
| [vlan-intervlan](vlan-intervlan/README.th.md) | VLAN, 802.1Q trunk, native VLAN mismatch, STP tie-breaker, router-on-a-stick, inter-VLAN routing ด้วย SVI | เสร็จแล้ว |

เครื่องมือ: GNS3, Cisco IOS 15.2 (7200), Cisco IOS 12.4 (3745 ที่ใส่ switch module NM-16ESW), VPCS, Wireshark

Repository นี้ไม่มีไฟล์ Cisco IOS image
