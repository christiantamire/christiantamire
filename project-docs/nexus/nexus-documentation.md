> Fictional scenario. Nexus Consulting Group does not exist. This documentation was built as a personal portfolio project in Cisco Packet Tracer.

Author: Musasizi Christian Samuel

Tool: Cisco Packet Tracer

Architecture: Collapsed two-tier campus network

Last updated: June 2026

> Security threat analysis, MITRE ATT&CK mappings, and NIST CSF gap analysis for this design are documented in [nexus-threat-model.md](./nexus-threat-model.md).

---

## Organisational profile

Nexus Consulting Group is a professional services firm based in Nakasero, Kampala. It provides management consulting, financial advisory, and HR solutions to mid-size organisations across East Africa. The firm runs from a single office with 70 staff across four departments.

Finance and HR handle sensitive data: payroll records, audit files, and personal staff information. That shaped the core network decision. Segment by department, force all cross-department traffic through explicit access control, and put internet-facing services in a separate zone the internal network can reach but the internet cannot reach through.

| Department | Staff count | Function |
| --- | --- | --- |
| IT | 10 | Infrastructure, internal systems, security operations |
| HR | 20 | Recruitment, payroll, staff records |
| Sales | 20 | Client acquisition, account management, CRM |
| Finance | 20 | Accounts, financial reporting, regulatory compliance |

---

## Physical inventory

| Hostname | Model | Role |
| --- | --- | --- |
| C-SW1 | Cisco Catalyst 3650-24PS | Core switch, HSRP active for VLANs 10/20/30, STP root for VLANs 10/20/30 |
| C-SW2 | Cisco Catalyst 3650-24PS | Core switch, HSRP active for VLANs 40/50, STP root for VLANs 40/50 |
| A-SW1 | Cisco Catalyst 2950T-24 | Access, SERVERS (VLAN 30) and IT (VLAN 10) |
| A-SW2 | Cisco Catalyst 2950T-24 | Access, HR (VLAN 20) and FINANCE (VLAN 50) |
| A-SW3 | Cisco Catalyst 2950T-24 | Access, SALES (VLAN 40) and HR (VLAN 20) |
| A-SW4 | Cisco Catalyst 2950T-24 | Access, FINANCE (VLAN 50) and SALES (VLAN 40) |
| ASA0 | Cisco ASA 5506 | Perimeter firewall, outside, two inside interfaces, DMZ |
| Router0 | Cisco 2811 | WAN edge, BGP toward ISP, OSPF toward ASA |
| DMZ-SW | Cisco Catalyst 2950T-24 | DMZ access switch |

Platform note on access switches: the 2950T-24 does not support `algorithm-type scrypt` for enable secret. That part is a genuine IOS version limitation. SSH, though, is supported by this platform; it simply isn't configured in the supplied access-switch template (no `crypto key generate`, no `ip domain-name`, no per-user `login local` on VTY). That's a configuration gap, not a hardware ceiling; see the management plane security section below for what that means in practice. Closing it doesn't require new hardware, just adding the missing SSH configuration to each access switch.

---

## PPDIOO mapping

Cisco's PPDIOO lifecycle model frames the project across six phases. Recording this maps the lab to the same methodology used in real network deployments.

| Phase | Work done in this project |
| --- | --- |
| Prepare | Defined org profile, department structure, data sensitivity requirements, and hardware budget constraints |
| Plan | Designed VLAN scheme, IP addressing, redundancy model, and security zone policy |
| Design | Produced logical and physical topology; selected protocols (RPVST+, HSRP, OSPF, LACP); defined ACL policy per zone |
| Implement | Configured all devices in Cisco Packet Tracer: core switches, access switches, ASA firewall, router |
| Operate | Verified OSPF adjacency, HSRP failover, STP convergence, and end-to-end connectivity |
| Optimise | Resolved multi-area OSPF instability; corrected STP/HSRP misalignment; tuned HSRP load balancing split |

---

## Physical topology

Collapsed two-tier design with no dedicated distribution layer. Both core switches connect to all four access switches via dual uplinks. Both core switches connect directly to the ASA on separate inside interfaces. The ASA connects to Router0 on its outside interface. The DMZ switch hangs off a dedicated ASA interface.

```
ISP
 |
Router0  Fa0/0: public IP  |  Fa0/1: 10.0.0.10/30
 |
ASA0
  OUTSIDE:         10.0.0.9/30   (g1/4)
  INTERNAL-CSW1:   10.0.0.2/30   (g1/1) -> C-SW1 g1/0/1: 10.0.0.1
  INTERNAL-CSW2:   10.0.0.6/30   (g1/2) -> C-SW2 g1/0/1: 10.0.0.5
  DMZ:             20.0.0.1/24   (g1/3) -> DMZ-SW -> Web, FTP, DNS servers
         |
    C-SW1 <---- EtherChannel Po1 (g1/0/6-7) ----> C-SW2
    g1/0/2-5 trunks                               g1/0/2-5 trunks
    v    v    v    v                             v    v    v    v
  A-SW1 A-SW2 A-SW3 A-SW4
```

Port-by-port connection map:

| Local device | Local port | Remote device | Remote port | Link type |
| --- | --- | --- | --- | --- |
| C-SW1 | g1/0/1 | ASA0 | g1/1 (INTERNAL-CSW1) | Routed point-to-point, 10.0.0.0/30 |
| C-SW2 | g1/0/1 | ASA0 | g1/2 (INTERNAL-CSW2) | Routed point-to-point, 10.0.0.4/30 |
| ASA0 | g1/4 | Router0 | Fa0/1 | Routed point-to-point, 10.0.0.8/30 |
| ASA0 | g1/3 | DMZ-SW | uplink | DMZ access, 20.0.0.0/24 |
| Router0 | Fa0/0 | ISP | - | WAN edge, public IP |
| C-SW1 | g1/0/2-5 | A-SW1 / A-SW2 / A-SW3 / A-SW4 | uplink (one trunk port per access switch) | 802.1Q trunk |
| C-SW2 | g1/0/2-5 | A-SW1 / A-SW2 / A-SW3 / A-SW4 | uplink (one trunk port per access switch) | 802.1Q trunk |
| C-SW1 | g1/0/6-7 | C-SW2 | g1/0/6-7 | LACP EtherChannel, Po1, trunk |
| A-SW1 to A-SW4 | g0/1-2 | C-SW1 and C-SW2 | g1/0/2-5 | Dual-homed 802.1Q trunk uplinks |
| A-SW1 | f0/1-2 | SERVERS endpoints | - | Access, VLAN 30 |
| A-SW1 | f0/3-4 | IT endpoints | - | Access, VLAN 10 |
| A-SW2 | f0/1-2 | HR endpoints | - | Access, VLAN 20 |
| A-SW2 | f0/3-4 | FINANCE endpoints | - | Access, VLAN 50 |
| A-SW3 | f0/1-2 | SALES endpoints | - | Access, VLAN 40 |
| A-SW3 | f0/3-4 | HR endpoints | - | Access, VLAN 20 |
| A-SW4 | f0/1-2 | FINANCE endpoints | - | Access, VLAN 50 |
| A-SW4 | f0/3-4 | SALES endpoints | - | Access, VLAN 40 |

Each access switch uplinks to both core switches, one trunk port to each. That's what gives RPVST+ and the dual-active HSRP split something to actually load-share across. If an access switch only had a single uplink, the per-VLAN STP split would have no second path to use.

---

## Logical topology

### Layer 2

**VLANs**

VLAN segmentation puts each department in its own broadcast domain. This limits ARP flood and DHCP attack blast radius to a single VLAN, and forces all cross-department traffic through the Layer 3 routing decision on the core switches, where ACLs enforce the access policy.

| VLAN | Name | Purpose |
| --- | --- | --- |
| 10 | IT | IT department endpoints |
| 20 | HR | HR department endpoints |
| 30 | SERVERS | DHCP server, DNS, FTP server, company printer |
| 40 | SALES | Sales department endpoints |
| 50 | FINANCE | Finance department endpoints |
| 60 | PHONES | Reserved for Phase 2 voice VLAN. Already provisioned in trunk allowed-lists, DHCP snooping, and DAI on every switch, but no endpoints exist on it yet |

| Protocol / feature | Purpose |
| --- | --- |
| VTP (domain `nexus`) | Centralises VLAN database management. C-SW1 is the VTP server; C-SW2 and all access switches are clients. Adding a VLAN on C-SW1 propagates to every switch automatically. Two VTP servers in the same domain create a revision number conflict risk: whichever has the higher number silently overwrites the other's database, so C-SW2 runs as a client despite being a Layer 3 switch. |
| EtherChannel (LACP 802.3ad, Po1) | Bundles two physical links between C-SW1 and C-SW2 into Po1. Without it, STP would block one of the parallel links entirely. With it, both links carry traffic and either can fail without dropping the connection. LACP is the open standard rather than Cisco's PAgP, which matters if non-Cisco hardware is ever introduced. Both sides use `mode active` for deterministic negotiation. |
| RPVST+ | Runs a separate STP instance per VLAN. This allows the two uplinks from each access switch to be active simultaneously for different VLANs rather than one being blocked entirely. C-SW1 is STP root primary for VLANs 10, 20, 30 (and root secondary for 40, 50, 60). C-SW2 is STP root primary for VLANs 40, 50, 60 (and root secondary for 10, 20, 30). This split matches the HSRP active distribution: traffic from any VLAN flows up the STP forwarding uplink directly to its HSRP active gateway, with no unnecessary hop across the core trunk. |
| Port security (sticky MAC) | Maximum of 2 addresses per access port, violation action shutdown. The maximum of 2 accounts for a phone plus a PC on the same port. A third MAC triggers err-disabled state immediately. This stops MAC flooding and prevents undocumented devices from joining the network without physical port recovery. |
| PortFast and BPDU Guard | PortFast skips the 30-second STP listening/learning delay on access ports, since end devices don't send BPDUs and shouldn't wait for STP convergence. BPDU Guard shuts the port the moment any BPDU arrives. A BPDU on an access port means a switch was connected where a host should be: either a misconfiguration or an attempt to inject a rogue switch into the spanning tree. |
| DHCP snooping | Drops DHCP server responses arriving on untrusted ports. All access ports are untrusted by default; uplinks toward the core switches (and the Po1 port-channel interface itself) are trusted. Without this, a device on any access port can impersonate a DHCP server and hand out fake gateways or DNS servers to clients. `no ip dhcp snooping information option` is required on all switches, since Packet Tracer's relay breaks when option 82 is enabled. Exception: A-SW1 f0/1, the port the DHCP server itself sits on, is explicitly trusted, since an untrusted port would otherwise drop the real DHCP server's own replies. |
| Dynamic ARP Inspection (DAI) | Validates ARP replies against the DHCP snooping binding table, using `validate dst-mac ip src-mac` checks. A reply claiming ownership of an IP that doesn't match the binding table is dropped. This stops ARP spoofing: an attacker on the same VLAN cannot redirect traffic by sending gratuitous ARPs with a spoofed MAC. |

STP root and HSRP active must match per VLAN. Misalignment was the root cause of the most significant issue encountered during the build; see the challenges section.

Platform limitation: ARP ACLs for static IP devices (servers, printer) are not supported in Packet Tracer. Those ports are handled by trusting at the interface level, which is less granular than production practice.

---

### Layer 3

| Protocol / feature | Purpose |
| --- | --- |
| Inter-VLAN routing via SVIs | Both core switches run `ip routing` with SVIs per VLAN. Cross-department traffic is routed at the core switch, not sent to the firewall first. The firewall only sees traffic crossing a security zone boundary. This keeps inter-department routing fast and keeps the firewall's job focused on what it's actually for. |
| HSRP with per-VLAN load balancing | Each VLAN has a virtual IP that hosts use as their default gateway. The virtual IP floats between switches. Rather than making C-SW1 active for everything, the load is split: C-SW1 owns VLANs 10/20/30, C-SW2 owns VLANs 40/50. Both switches carry active traffic at the same time, which distributes load and keeps both devices doing useful work rather than one sitting idle in standby. Preempt is configured on the intended active switch only. Setting it on both causes the recovering switch to steal the active role mid-session after a failover, which disrupts established connections. |
| OSPF (process 15, single Area 0) | Handles dynamic routing between core switches, ASA, and Router0. All devices are configured in Area 0. Multi-area OSPF was tested first and caused 25 to 75% sustained packet loss; see the challenges section. Single area eliminates the ABR role, Type 3 LSA conflicts, and the interaction between OSPF reconvergence and HSRP failover. Router0 uses `default-information originate always` to inject a default route into the OSPF domain, so internal devices get one exit path: toward Router0 via the ASA. |
| Floating static default routes | Both core switches configure `ip route 0.0.0.0 0.0.0.0 <ASA inside IP>` as a backup path. Implementation gap: the design intent is for this static to sit at administrative distance 254, below OSPF's AD of 110, so it stays dormant unless OSPF fails. The commands as configured (`ip route 0.0.0.0 0.0.0.0 10.0.0.2` on C-SW1, `ip route 0.0.0.0 0.0.0.0 10.0.0.6` on C-SW2) do not specify a distance, which means each takes the IOS default AD of 1, more preferred than OSPF, not less. As implemented, these statics would always win over the OSPF-learned default route rather than acting as a floating backup. This needs `ip route 0.0.0.0 0.0.0.0 <next-hop> 254` to match the documented design. |
| Loopback interfaces (design intent) | The design calls for a /32 loopback per core switch (C-SW1: `1.2.1.2/32`, C-SW2: `2.1.2.1/32`) to anchor a stable OSPF router ID and to give IT a fixed SSH management target that survives interface flaps. Implementation gap: the supplied configuration sets `router-id 1.2.1.2` / `router-id 2.1.2.1` directly under the OSPF process, but no `interface loopback` is actually created on either switch. The router ID is set as a value, but there is no loopback interface to advertise or to SSH into. The loopback-reachability part of the design isn't yet implemented. |
| DHCP relay (`ip helper-address`) | The DHCP server is at 192.168.3.4 in VLAN 30. SVIs for VLANs 10, 20, 40, and 50 on both core switches relay broadcast DHCP requests as unicasts to that address. VLAN 30 has no relay configured, since relaying a DHCP request back to the server that's already on the same subnet would cause a relay loop. |
| Inter-VLAN ACLs | Applied on VLAN SVIs on both core switches, in both directions; see the dedicated ACL section below for the full outbound/inbound breakdown. Both switches carry identical ACLs because either switch may route traffic for any VLAN depending on HSRP active state. An ACL on C-SW1 only does nothing for traffic routed through C-SW2. |
| NAT/PAT (ASA) | The ASA translates internal RFC1918 addresses to its outside interface IP before forwarding internet-bound traffic toward Router0. Each inside interface has its own NAT object (`INTERNAL-VLANS` for C-SW1's link, `INTERNAL-VLANS-1` for C-SW2's link) pointing to the same outside interface via dynamic PAT. DMZ servers also NAT outbound (object `DMZ`) for internet access: OS updates, external API calls. |
| ASA static default route | ASA0 has its own static default, `route OUTSIDE 0.0.0.0 0.0.0.0 10.0.0.10`, pointing to Router0. This is the firewall's own path off the internal network. It is separate from, and sits alongside, the OSPF process running between the ASA and the core switches. |

**Inter-VLAN ACL policy, full detail**

The access control policy follows the principle of least privilege. Departments are isolated from each other by default, with IT and SERVERS explicitly permitted as the only trusted sources. The deny rule uses a summary wildcard `192.168.0.0 0.0.7.255` to catch any internal source not explicitly permitted above it, which is cleaner than listing each department VLAN individually.

ACLs are applied in two directions per VLAN SVI. This governs who can reach a department (`out`, evaluated as traffic leaves toward that VLAN) and what that department can reach elsewhere (`in`, evaluated as traffic enters the SVI from the local VLAN):

| VLAN | ACL applied `out` (who may reach this VLAN) | ACL applied `in` (what this VLAN may reach) |
| --- | --- | --- |
| 30, SERVERS | ACL 100: IT permitted ICMP and all IP; all other depts denied ICMP, all IP permitted | ACL 120: SERVERS permitted ICMP to IT; ICMP to any other internal dept denied; all IP permitted |
| 20, HR | ACL 101: IT and SERVERS full IP access; all other depts denied | ACL 110: HR permitted full IP to IT and SERVERS; denied to any other internal dept; all other IP permitted (i.e., internet) |
| 40, SALES | ACL 102: IT and SERVERS full IP access; all other depts denied | ACL 111: SALES permitted full IP to IT and SERVERS; denied to any other internal dept; all other IP permitted |
| 50, FINANCE | ACL 103: IT and SERVERS full IP access; all other depts denied | ACL 112: FINANCE permitted full IP to IT and SERVERS; denied to any other internal dept; all other IP permitted |
| 10, IT | No outbound ACL. IT requires unrestricted access for network management and support operations. | No inbound ACL, same rationale |

In practice the `in`-direction ACLs (120, 110, 111, 112) are what actually enforce department isolation from the sending side. They stop HR, Sales, or Finance from initiating traffic toward each other, while the `out`-direction ACLs (100 to 103) enforce it from the receiving side as a second layer. Both are needed because either core switch can be the one routing the traffic depending on which one is HSRP-active for the destination VLAN at that moment.

ACL 100/120 on VLAN 30 additionally narrows ICMP specifically. IT can ping SERVERS and SERVERS can ping IT, which is useful for diagnostics, but ICMP between SERVERS and any other department is denied, while non-ICMP IP traffic (DHCP relay responses, print jobs, service traffic) is left unaffected in both directions.

OSPF wildcard note: the statement `network 192.168.0.0 0.0.7.255 area 0` covers subnets .0.x through .7.x in a single line. The subnets .6.x and .7.x don't exist on any interface, so OSPF never advertises them. The wildcard matches the addressing cleanly without per-subnet entries.

Design note flagged for review, DMZ subnet in OSPF: the design intent is for the DMZ subnet to be excluded from OSPF, so that internal switches cannot learn a route to the DMZ and bypass the firewall's zone enforcement. The ASA's actual OSPF configuration, however, advertises `network 20.0.0.0 255.255.255.0 area 0` alongside its two inside transit links. As configured, the DMZ subnet is being injected into the OSPF domain, which contradicts the stated design goal and should be corrected. See open items.

---

## Perimeter security

**ASA zone model**

| Interface | Name | Security level | Network | Connected to |
| --- | --- | --- | --- | --- |
| g1/1 | INTERNAL-CSW1 | 100 | 10.0.0.0/30 | C-SW1 g1/0/1 |
| g1/2 | INTERNAL-CSW2 | 100 | 10.0.0.4/30 | C-SW2 g1/0/1 |
| g1/3 | DMZ | 50 | 20.0.0.0/24 | DMZ-SW |
| g1/4 | OUTSIDE | 0 | 10.0.0.8/30 | Router0 Fa0/1 |

Known limitation, dual inside interfaces: both inside interfaces sit at security level 100. The command `same-security-traffic permit inter-interface` is required for traffic to pass between them and is not available in Packet Tracer's ASA 5506 implementation. This topology was chosen to simulate the budget reality of a mid-size firm that cannot afford a dedicated inside transit switch. In production, a single inside interface connected to a small transit switch would be correct, with HSRP redundancy handled transparently below it.

**ASA ACL policy**

Internal hosts reach DMZ servers on specific permitted ports only (`INTERNAL_IN`, applied inbound on both inside interfaces). A deny rule for the DMZ subnet fires before the broad internet permit, so hosts cannot reach DMZ servers on unpermitted ports even though the final rule allows all other outbound traffic.

DMZ servers reach the internet for updates and external service calls (`DMZ_IN`). They cannot initiate connections to internal hosts beyond what's explicitly permitted. The ASA's stateful inspection handles legitimate return traffic automatically, and an explicit deny blocks DMZ-to-internal traffic before the broad internet permit.

External traffic reaches only the three DMZ servers on their published ports (`OUTSIDE_IN`). No external traffic reaches internal hosts directly.

---

## Management plane security

Core switches and access switches implement materially different levels of management plane control. Some of this is a genuine platform constraint of the 2950T-24 (no scrypt support); the SSH gap, however, is a configuration omission rather than a hardware ceiling.

**Core switches (C-SW1, C-SW2)**

| Username | Privilege | Access |
| --- | --- | --- |
| admin | 15 | Full IOS access including configuration mode |
| support | 1 | Show commands only |

SSH version 2 replaces Telnet for all remote management. RSA key generation at 2048 bits provides the key material; `transport input ssh` under VTY explicitly blocks Telnet. Both console and VTY lines use `login local`, authenticating against the local username database. VTY ACL (ACL 90) restricts SSH access via `access-class 90 in`, permitting only the IT subnet (192.168.1.0/24). Enable secret uses scrypt-based hashing (`algorithm-type scrypt`). `service password-encryption` encrypts all plaintext passwords in the running config; this is reversible and is not a strong control — it prevents casual shoulder-surfing rather than protecting against a determined attacker with config file access.

**Access switches (A-SW1 to A-SW4)**

SSH not enabled. The 2950T-24 does support SSH, but the supplied template doesn't configure it: no `crypto key generate`, no `ip domain-name`, no per-user `login local` on VTY. Management is via Telnet only. Console uses a shared plaintext line password (`cisco-nexus`). VTY uses a shared line password (`cisco-nexus-123`) with plain `login`, no `access-class`. Enable secret uses standard MD5 hashing (no scrypt support on this platform). `service password-encryption` is present, same caveat as above.

This gap is the most significant difference between the intended management-plane posture and what's actually running on the access switches. Unlike the scrypt limitation, this one doesn't require new hardware. Enabling SSH on the existing 2950T-24s would close it directly.

---

## IP and VLAN planning

**Internal subnets**

| VLAN | Subnet | Gateway (HSRP VIP) | DHCP served |
| --- | --- | --- | --- |
| 10, IT | 192.168.1.0/24 | 192.168.1.1 | Yes |
| 20, HR | 192.168.2.0/24 | 192.168.2.1 | Yes |
| 30, SERVERS | 192.168.3.0/24 | 192.168.3.1 | No, all static |
| 40, SALES | 192.168.4.0/24 | 192.168.4.1 | Yes |
| 50, FINANCE | 192.168.5.0/24 | 192.168.5.1 | Yes |
| 60, PHONES (reserved) | 192.168.6.0/24 (reserved, unused) | 192.168.6.1 (reserved, unused) | Planned, Phase 2 |

**HSRP per-VLAN summary**

| VLAN | C-SW1 SVI | C-SW2 SVI | Virtual IP | Active switch |
| --- | --- | --- | --- | --- |
| 10 | 192.168.1.2 | 192.168.1.3 | 192.168.1.1 | C-SW1 |
| 20 | 192.168.2.2 | 192.168.2.3 | 192.168.2.1 | C-SW1 |
| 30 | 192.168.3.2 | 192.168.3.3 | 192.168.3.1 | C-SW1 |
| 40 | 192.168.4.2 | 192.168.4.3 | 192.168.4.1 | C-SW2 |
| 50 | 192.168.5.2 | 192.168.5.3 | 192.168.5.1 | C-SW2 |

**WAN links**

| Link | Subnet | Device A | Device B |
| --- | --- | --- | --- |
| C-SW1 to ASA | 10.0.0.0/30 | C-SW1: 10.0.0.1 | ASA: 10.0.0.2 |
| C-SW2 to ASA | 10.0.0.4/30 | C-SW2: 10.0.0.5 | ASA: 10.0.0.6 |
| ASA to Router0 | 10.0.0.8/30 | ASA: 10.0.0.9 | Router0: 10.0.0.10 |

**VLAN 30 static assignments**

| Device | IP |
| --- | --- |
| DHCP server | 192.168.3.4 |
| Company printer | 192.168.3.7 |

**DMZ static assignments**

| Device | IP | External access | Internal access |
| --- | --- | --- | --- |
| Web server | 20.0.0.18 | 80, 443 | 80, 443 |
| FTP server | 20.0.0.12 | 20, 21 | 20, 21 |
| DNS server | 20.0.0.5 | 53 (TCP/UDP) | 53 (TCP/UDP) |

Note: 20.0.0.0/24 is a public IP range used here as a simulation choice. Production DMZ addressing would use RFC1918 space or a properly allocated public range.

VTP domain: all switches share VTP domain `nexus` (C-SW1 server, everything else client).

Design decision, FTP in DMZ: FTP was placed in the DMZ rather than VLAN 30 to allow external partners to retrieve shared resources without entering the internal network. Internal users reach it through the ASA on ports 20 and 21. Production hardening note: standard FTP transmits credentials and file data in plaintext. In production, SFTP (SSH File Transfer Protocol, port 22) would replace FTP entirely. Packet Tracer does not support SFTP, so FTP is used here as a simulation constraint.

---

## Challenges faced

**HSRP standby persisting on intended active switch**

C-SW1 was configured with HSRP priority 110 and preempt but remained in standby. C-SW2 was winning the active election despite priority 100.

Root cause: STP root was not configured explicitly. STP elected C-SW2 as root bridge based on lower bridge ID. C-SW1's access-facing ports went into alternate/blocking state. With those ports blocked, HSRP hellos from C-SW1 couldn't reach hosts and the election behaviour was wrong.

Fix: `spanning-tree vlan 10,20,30 root primary` on C-SW1. Once STP converged with C-SW1 as root, its ports forwarded and HSRP immediately moved the active role to C-SW1.

Lesson: STP root and HSRP active must be configured on the same switch per VLAN. HSRP priority alone doesn't determine the active switch if the underlying Layer 2 path is broken.

**Multi-area OSPF instability**

Configuring VLANs in Area 1 and WAN links in Area 0 caused 25 to 75% sustained packet loss across all internal traffic.

Root cause: C-SW1 and C-SW2 both became ABRs. Both advertised identical Area 1 prefixes into Area 0 as Type 3 LSAs simultaneously. With equal costs on both paths, the ASA and Router0 oscillated between them. HSRP failover events triggered simultaneous OSPF reconvergence across both areas, compounding the loss window.

Alternative considered: tuning HSRP delay timers to align with OSPF convergence using `standby delay minimum/reload`. Rejected, since slowing HSRP failover to match OSPF convergence defeats the purpose of sub-second gateway redundancy.

Fix: collapsed all devices into single Area 0. No ABR role, no Type 3 LSA generation, no inter-area reconvergence. Packet loss dropped to zero.

Lesson: multi-area OSPF adds value at scale with hundreds of routers. For a collapsed two-tier campus with five subnets, single Area 0 is the correct design. HSRP and ABR roles conflict when on the same device. HSRP failover events trigger OSPF reconvergence at the area boundary simultaneously.

**EtherChannel not forming**

`channel-group` entered before `channel-protocol lacp` and without `mode active`. Port-channel did not come up.

Fix: correct order is `channel-protocol lacp` then `channel-group 1 mode active`.

**SVIs administratively down**

Packet Tracer defaults SVIs to administratively down. HSRP would not form until `no shut` was added under every VLAN interface on both core switches.

**HSRP group number missing**

Standby commands entered without group numbers. All VLANs defaulted to HSRP group 0. Each VLAN needs its own group number matching the VLAN ID.

**Preempt on standby switch**

Preempt was configured on both switches. This risks the standby switch stealing the active role back mid-session after a failover event. Preempt belongs only on the intended active switch.

**same-security-traffic not available in Packet Tracer**

Two inside interfaces at security level 100 require `same-security-traffic permit inter-interface`. The command is not supported in the Packet Tracer ASA 5506 implementation. Documented as a platform limitation.

---

## Open items identified during configuration review

These were found while cross-checking the running configuration against the design intent described above. They aren't yet resolved and are listed separately from "Challenges faced" because they're configuration gaps caught on review, not issues hit and fixed during the build.

1. Floating static default routes have no administrative distance set. As configured, they default to AD 1 (more preferred than OSPF's 110), which would make them primary rather than a backup. Needs `254` appended to both `ip route` commands.
2. DMZ subnet is advertised into OSPF by the ASA, despite the documented design intent being to keep it out of the routing domain. Needs either removal of `network 20.0.0.0 255.255.255.0 area 0` from the ASA's OSPF process, or an explicit decision to keep it with the rationale documented.
3. Loopback interfaces don't exist yet. `router-id` is set as a bare value on both core switches' OSPF process, but no `interface loopback0` was created. Needs the loopback interfaces actually created, addressed, and added to the OSPF network statement.
4. VTY ACL (90) is a standard ACL. It correctly restricts SSH source to the IT subnet, but cannot enforce destination-specific granularity. Would need an extended ACL referencing destination addresses and port 22 to match the design as described.
5. SSH is not configured on the access switches, though the platform supports it. Closing it needs `ip domain-name`, `crypto key generate rsa`, `login local` with per-user accounts, and `transport input ssh` added to each access switch's VTY configuration.

---

## Planned additions

| Item | Notes |
| --- | --- |
| BGP on Router0 | eBGP toward ISP. No BGP-to-OSPF redistribution; `default-information originate always` handles the default route. |
| End-to-end verification | HSRP failover test (pull cable on C-SW1), DHCP lease verification per VLAN, ping matrix across all VLANs, OSPF neighbour verification. |
| Voice VLAN 60, Phase 2 | VLAN 60 PHONES designed. Physical phones removed from current topology; no dedicated CME router within budget. Deferred. |
| DHCP snooping and DAI verification | Test rogue DHCP server detection and ARP spoofing drop behaviour explicitly. |
| Centralised logging (syslog) | Add a syslog server in the SERVERS VLAN and `logging host <address>` on every switch and the ASA. Cheapest, highest-value first step into the CSF Detect function. Currently port security violations, BPDU Guard triggers, and DAI drops are visible only on local device consoles. |
