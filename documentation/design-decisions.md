# Design Decisions — NovaFin Enterprise Network

This document details the reasoning behind key architectural choices made during the design and implementation of the NovaFin campus network.

## 1. Hierarchical vs. Flat Network Design

**Decision:** Adopted Cisco's 3-tier hierarchical model (Access → Core → Edge).

**Alternatives considered:**
- Flat Layer 2 design with a single core switch
- 2-tier collapsed core design

**Rationale:**
- Scalability: The design supports growth to 200+ users without architectural change
- Fault isolation: Problems in one access switch don't cascade to other segments
- Simpler troubleshooting: Clear demarcation of Layer 2 vs Layer 3 responsibilities
- Industry standard: Aligns with real enterprise design practices

## 2. Static Routing vs. Dynamic Routing (OSPF/EIGRP)

**Decision:** Used static routes between SW-Core and R1.

**Alternatives considered:**
- OSPF (single area)
- EIGRP (Cisco proprietary)

**Rationale:**
- Only 2 routers in the topology — no dynamic protocol overhead justified
- Predictable routing behavior for troubleshooting
- Lower CPU/memory footprint
- **Future migration path:** OSPF would be added when the network grows beyond 3-4 routers

## 3. NAT/PAT Placement on R1

**Decision:** PAT (NAT overload) configured on R1's WAN interface using the entire 172.16.0.0/16 aggregate.

**Alternatives considered:**
- Static NAT for specific services (rejected — no public-facing services)
- Dynamic NAT with pool (rejected — only single public IP available)

**Rationale:**
- Single public IP maximizes cost efficiency (typical for SMB internet contracts)
- Aggregate /16 in ACL means new VLANs automatically get NAT without config changes

## 4. Guest VLAN Design

**Decision:** Guest VLAN (30) has its own subnet, DHCP pool with shorter lease, and dedicated ACL for isolation.

**Alternatives considered:**
- Separate physical infrastructure for guests (over-engineered for this size)
- Guest traffic on shared VLAN with employee traffic (security risk)

**Rationale:**
- Logical separation via VLAN is standard practice
- Short lease (1 day vs 7 days for employees) accommodates high guest turnover
- Dedicated ACL enforces isolation from internal networks while permitting internet access

## 5. Port Security Configuration Choices

**Decision:** Sticky MAC with `violation shutdown` for employee ports, `violation restrict` for guest ports.

**Rationale:**
- Employee ports: Zero-tolerance — unauthorized device = port shutdown, admin notification
- Guest ports: More permissive — guest devices change frequently; shutdown would create excessive tickets
- Voice ports (IP phone + PC): `maximum 2` accommodates both devices, `violation restrict` prevents voice service disruption

## 6. Management VLAN Isolation

**Decision:** VLAN 99 dedicated for network device management, with static IPs and no DHCP.

**Rationale:**
- Management traffic isolated from user traffic (security and performance)
- Static IPs on switches ensure predictable access even during DHCP failure
- Extended ACL blocks user VLANs from reaching management VLAN

## 7. STP Root Bridge Selection

**Decision:** SW-Core hard-coded as primary root bridge (priority 4096). SW-Access-1 as secondary root (priority 8192).

**Rationale:**
- Predictable topology — root bridge election isn't left to MAC address chance
- SW-Core is architecturally the highest-bandwidth switch; making it root aligns STP tree with traffic patterns
- Secondary root ensures graceful failover if SW-Core requires maintenance

## 8. Banner MOTD Legal Notice

**Decision:** Login banner explicitly warns of unauthorized access consequences.

**Rationale:**
- Legal requirement in many jurisdictions to display before login
- Removes "I didn't know" defense in prosecution scenarios
- Industry standard for enterprise networks

---

## Trade-offs Acknowledged

**Single Point of Failure at SW-Core:**
This design has SW-Core as the only Layer 3 device. In production, this would be addressed with a redundant Layer 3 switch and HSRP/VRRP between SVIs. For this design's scope (SMB), the trade-off between cost and redundancy favors single-switch simplicity.

**No Firewall Between R1 and Internet:**
For simulation purposes, R1 directly connects to ISP. In production, a firewall (Cisco ASA, Palo Alto, or Fortinet) would sit between R1 and the internet edge, with R1 handling only routing and NAT.

**No IPS/IDS:**
No intrusion detection or prevention is deployed. In a real fintech environment, this would be non-negotiable.

**No High Availability for R1:**
A single edge router is a SPOF for internet access. Production would deploy redundant routers with HSRP.

These acknowledged limitations represent conscious trade-offs for the design's scope (SMB, budget-conscious, simulation environment), not oversights.