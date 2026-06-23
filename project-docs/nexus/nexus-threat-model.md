> Fictional scenario. Nexus Consulting Group does not exist. This document was produced as part of a personal portfolio project.

Author: Musasizi Christian Samuel
Last updated: June 2026
Companion document: `nexus-documentation.md`

---

## Purpose

This document answers the question the network documentation does not: is the Nexus Consulting Group network actually secure, and against what threats? Where `nexus-documentation.md` records what was built and how, this document records what the network is defending against, what it successfully defends, and where it falls short.

The analysis uses two frameworks together because they answer different questions. MITRE ATT&CK maps specific adversary techniques to specific controls — it answers "does this control stop that attack?" NIST CSF 2.0 maps the full security lifecycle — it answers "across everything a security program should do, what is actually covered?"

---

## Organisational context and asset classification

Nexus Consulting Group is a 70-person professional services firm. The assets worth protecting, and the reason they matter:

| Asset | Sensitivity | Rationale |
| --- | --- | --- |
| Finance VLAN (VLAN 50) | High | Payroll data, accounts, regulatory filings. Exposure creates direct financial and legal liability. |
| HR VLAN (VLAN 20) | High | Personal staff records, recruitment data. Subject to data protection obligations. |
| SERVERS VLAN (VLAN 30) | Medium-High | DHCP server, internal DNS, company printer. Compromise of DHCP or DNS enables redirection attacks across all VLANs. |
| DMZ servers | Medium | Web, FTP, DNS servers. Internet-facing; compromise could pivot inward or damage firm reputation. |
| Core switches (C-SW1, C-SW2) | Critical | Compromise of either core switch gives an attacker control over routing, ACLs, and VLAN policy for the entire network. |
| ASA firewall | Critical | The sole enforcement point between internal zones and the internet. Bypass eliminates perimeter controls entirely. |
| IT VLAN (VLAN 10) | Medium | Administrative access subnet. Compromise gives attacker SSH access to network infrastructure. |

---

## Threat actors

For a mid-size professional services firm of this profile, three threat actor categories are realistic:

**External opportunistic attacker** — automated scanning and exploitation of internet-facing services. Targets: DMZ web and FTP servers, any open management ports reachable from the outside. Motivation: data theft, ransomware, botnet recruitment. Low sophistication, high volume.

**Targeted external attacker** — deliberate reconnaissance against a consulting firm's client data or financial records. Motivation: competitive intelligence, financial fraud, extortion. Higher sophistication, willing to chain multiple techniques.

**Malicious or negligent insider** — a staff member (Finance, HR, Sales) who intentionally or accidentally accesses systems beyond their authorisation. Motivation varies; the threat is horizontal (lateral movement between departments) rather than perimeter-focused.

No nation-state or APT threat is assumed at this organisational scale.

---

## Attack scenarios and control mapping

### Scenario 1 — Rogue DHCP server (insider or compromised endpoint)

An attacker plugs a device into an access port and runs a DHCP server, handing out a fake default gateway or DNS server to clients on the same VLAN. Clients that accept the fake lease route all traffic through the attacker.

| | |
|---|---|
| MITRE technique | T1557.003 — DHCP spoofing |
| Tactic | Credential access / adversary-in-the-middle |
| Affected assets | Any VLAN where the rogue device gains access |
| Control implemented | DHCP snooping on all access switches. All access ports untrusted by default; only uplinks toward core switches are trusted. DHCP server responses on untrusted ports are dropped before they reach clients. |
| Residual risk | Low. DHCP snooping directly and completely addresses this technique. |

---

### Scenario 2 — ARP cache poisoning (insider or compromised endpoint)

An attacker on the same VLAN broadcasts gratuitous ARP replies claiming ownership of another device's IP (e.g., the default gateway). Clients update their ARP tables and send traffic to the attacker instead.

| | |
|---|---|
| MITRE technique | T1557.002 — ARP cache poisoning |
| Tactic | Credential access / adversary-in-the-middle |
| Affected assets | Any VLAN the attacker has access to |
| Control implemented | Dynamic ARP Inspection (DAI) validates every ARP reply against the DHCP snooping binding table. A reply claiming an IP not matching the binding is dropped. `validate dst-mac ip src-mac` checks are enabled. |
| Residual risk | Low for DHCP clients. Static IP devices (VLAN 30 servers, printer) are not covered by the DHCP binding table; Packet Tracer does not support ARP ACLs for static entries, so those ports rely on interface-level trust instead. This is a platform limitation, not a design choice, and is documented as such. |

---

### Scenario 3 — Rogue switch / STP root bridge takeover

An attacker connects an unauthorised switch to an access port and sends superior BPDUs, causing STP to elect their device as the root bridge. This repositions the attacker's switch in the forwarding path for some or all VLANs, enabling traffic interception.

| | |
|---|---|
| MITRE technique | T1200 — Hardware additions (rogue switch) |
| Tactic | Initial access / adversary-in-the-middle |
| Affected assets | All VLANs; STP topology is network-wide |
| Control implemented | BPDU Guard on all access ports. The moment a BPDU arrives on an access port, the port is placed into err-disabled state. An attacker cannot send BPDUs from an access port without triggering immediate shutdown. Port security (sticky MAC, max 2, violation shutdown) adds a second layer: an unauthorised device triggers a violation before it can participate in STP at all. |
| Residual risk | Low. BPDU Guard is the correct and complete control. The residual gap is detection: the port goes err-disabled correctly, but no centralised alert fires. An attacker's attempt is blocked, but nobody is notified it happened unless someone is watching the switch console. See the Detect gap in the NIST section. |

---

### Scenario 4 — MAC flooding (CAM table exhaustion)

An attacker floods the switch with frames using random source MAC addresses, filling the CAM table. Once full, the switch falls back to flooding all frames out all ports in the VLAN, allowing the attacker to capture traffic not addressed to them.

| | |
|---|---|
| MITRE technique | T1200 — Hardware additions; T1557 — adversary-in-the-middle (via flooding) |
| Tactic | Collection / credential access |
| Affected assets | Any VLAN the attacker has physical access to |
| Control implemented | Port security (sticky MAC) limits each access port to 2 MAC addresses. A third MAC triggers err-disabled immediately, stopping the flood before the CAM table can be exhausted. |
| Residual risk | Low. |

---

### Scenario 5 — OSPF LSA injection (routing plane attack)

An attacker on the network segment crafts OSPF Hello packets to form an unauthorised adjacency with a core switch, then injects LSAs advertising more-specific prefixes (e.g. a /29 within a legitimate /26). The longest-prefix match rule causes routers to forward traffic for those hosts toward the attacker's device rather than the legitimate destination — a silent man-in-the-middle at the routing layer.

This attack is well-documented. Researcher Gabi Nakibly demonstrated persistent LSA injection at Black Hat 2011 against Cisco IOS 15.x, showing that uncompromised routers can be tricked into propagating false LSAs without triggering OSPF's fight-back mechanism.

| | |
|---|---|
| MITRE technique | T1557 — Adversary-in-the-middle; T1599 — Network boundary bridging |
| Tactic | Adversary-in-the-middle / collection |
| Affected assets | Any traffic passing through core switches; routing table integrity |
| Control implemented | **None currently.** OSPF is running without MD5 authentication. Any device that can reach the OSPF multicast address (224.0.0.5) on the segment can attempt adjacency formation. |
| Residual risk | **High.** This is the most significant unmitigated routing-plane risk in the current design. |
| Remediation | `area 0 authentication message-digest` on all OSPF-participating devices (C-SW1, C-SW2, ASA, Router0), with matching `ip ospf message-digest-key` on each interface. Additionally, access-facing interfaces should be configured as OSPF passive interfaces (`passive-interface <access-facing SVI>`) so end devices cannot attempt OSPF participation even if they craft Hello packets. |

---

### Scenario 6 — HSRP gateway hijacking

An attacker crafts HSRP Hello messages with a higher priority than the legitimate active router, causing HSRP to elect the attacker's device as the active gateway. All traffic from the affected VLAN is then forwarded through the attacker.

| | |
|---|---|
| MITRE technique | T1557 — Adversary-in-the-middle |
| Tactic | Adversary-in-the-middle |
| Affected assets | All traffic from the hijacked VLAN's hosts |
| Control implemented | Partial. Inter-VLAN ACLs prevent non-IT VLANs from reaching SVI and HSRP IP addresses directly, which limits the attack surface. An attacker on VLAN 20 (HR) cannot reach the HSRP virtual IP from their segment due to the inbound ACL denying cross-department internal traffic. |
| Residual risk | Medium. HSRP MD5 authentication is not configured. A device that can reach the HSRP multicast address on the segment (e.g., a compromised IT endpoint, or a device on the same VLAN as the HSRP speakers) could attempt the attack. Non-IT VLANs are partially protected by ACL policy, but HSRP authentication would be the direct and complete control. |
| Remediation | `standby <group> authentication md5 key-string <key>` on each HSRP group on both core switches. |

---

### Scenario 7 — Lateral movement between departments (insider)

A Finance staff member attempts to access HR systems, or a compromised Sales endpoint attempts to pivot to Finance. Cross-department traffic is the primary insider threat vector for a consulting firm with sensitive HR and Finance data.

| | |
|---|---|
| MITRE technique | T1021 — Remote services (lateral movement) |
| Tactic | Lateral movement |
| Affected assets | HR VLAN (VLAN 20), Finance VLAN (VLAN 50) |
| Control implemented | Inter-VLAN ACLs on both core switches in both directions. Finance cannot initiate traffic to HR, Sales, or any other department. HR is likewise isolated. Only IT and SERVERS can reach any department. ACLs are applied inbound (from the source VLAN) and outbound (toward the destination VLAN), providing two enforcement points regardless of which core switch is routing the traffic. |
| Residual risk | Low for network-layer lateral movement. No east-west IDS exists, so unusual traffic patterns within permitted flows (e.g., a SERVERS endpoint being used as a pivot) would not be detected. |

---

### Scenario 8 — Exploitation of public-facing DMZ services

An external attacker targets the web server (port 80/443), FTP server (ports 20/21), or DNS server (port 53) from the internet, seeking to exploit a vulnerability in the service software.

| | |
|---|---|
| MITRE technique | T1190 — Exploit public-facing application |
| Tactic | Initial access |
| Affected assets | DMZ web, FTP, DNS servers |
| Control implemented | `OUTSIDE_IN` ACL on the ASA permits only specific ports to specific DMZ server IPs. All other inbound traffic from OUTSIDE is denied. The ACL is host-specific: web traffic only reaches 20.0.0.18, FTP only reaches 20.0.0.12, DNS only reaches 20.0.0.5. Non-standard ports and direct access to internal hosts are blocked entirely. |
| Residual risk | Medium. ACL policy limits the exposed surface area, but the DMZ servers themselves are Packet Tracer placeholders with no OS hardening, no patch management, and no WAF or IPS in front of them. In production, server hardening, a WAF for the web server, and an IPS would be necessary additions. |

---

### Scenario 9 — DMZ pivot to internal network

An attacker who compromises a DMZ server attempts to use it as a launch point to attack internal hosts.

| | |
|---|---|
| MITRE technique | T1210 — Exploitation of remote services (post-DMZ-compromise) |
| Tactic | Lateral movement |
| Affected assets | Internal VLANs, core switches |
| Control implemented | `DMZ_IN` ACL blocks DMZ servers from initiating connections to internal hosts. The ASA's zone model (DMZ at security level 50, INTERNAL at 100) enforces this at the firewall level in addition to the ACL. A compromised DMZ server cannot reach 192.168.x.x addresses. |
| Residual risk | Low for direct connection attempts. The DMZ subnet being inadvertently advertised into OSPF (see open items in nexus-documentation.md) is a related risk: internal switches could learn a route to the DMZ, potentially enabling return-path confusion. This should be corrected. |

---

### Scenario 10 — Brute force of management interfaces

An attacker with network access attempts to brute-force SSH credentials on the core switches, or Telnet credentials on the access switches.

| | |
|---|---|
| MITRE technique | T1110 — Brute force; T1078 — Valid accounts |
| Tactic | Credential access / privilege escalation |
| Affected assets | Core switches (C-SW1, C-SW2), access switches (A-SW1 to A-SW4) |
| Control implemented (core switches) | VTY ACL 90 restricts SSH attempts to the IT subnet (192.168.1.0/24) only. A device outside VLAN 10 cannot reach the SSH port on core switches at all. Per-user `login local` authentication produces an audit trail. SSHv2 with 2048-bit RSA. Scrypt-hashed enable secret. |
| Control implemented (access switches) | None meaningful. Telnet with a shared line password. Any device that can route to an access switch's management IP can attempt the shared password. No `access-class` restricting source subnets. |
| Residual risk | Low for core switches. High for access switches — this is the most significant management-plane gap. |

---

### Scenario 11 — Privilege escalation via shared credentials

An attacker who obtains the shared access-switch password (`cisco-nexus` / `cisco-nexus-123`) or the enable secret gains full control of those devices with no individual accountability.

| | |
|---|---|
| MITRE technique | T1078.003 — Local accounts |
| Tactic | Privilege escalation |
| Affected assets | Access switches A-SW1 to A-SW4 |
| Control implemented | Core switches use per-user `login local` with privilege level separation (admin/support). Access switches use shared passwords only — no per-user accounts, no privilege separation. |
| Residual risk | Medium. Mitigated on core switches; not mitigated on access switches. |

---

### Scenario 12 — CDP/LLDP topology reconnaissance

An attacker connected to any access port passively receives CDP (Cisco Discovery Protocol) advertisements from the connected switch, revealing device model, IOS version, IP addresses, and native VLAN. This is free intelligence for planning further attacks.

| | |
|---|---|
| MITRE technique | T1590 — Gather victim network information |
| Tactic | Reconnaissance |
| Affected assets | Network topology confidentiality |
| Control implemented | None. CDP is running by default on all interfaces. |
| Residual risk | Medium. An attacker on any access port learns the connected switch model, management IP, and platform version passively. `no cdp enable` on all access-facing ports (while keeping it on trunk/uplink ports for network management purposes) would close this. |

---

## MITRE ATT&CK mapping summary

| Control implemented | ATT&CK technique | Tactic | Mitigated? |
| --- | --- | --- | --- |
| DHCP snooping | T1557.003 — DHCP spoofing | Credential access | Yes |
| Dynamic ARP Inspection | T1557.002 — ARP cache poisoning | Credential access | Yes (DHCP clients); partial (static IPs) |
| Port security, sticky MAC | T1200 — Hardware additions; T1557 via MAC flood | Initial access / collection | Yes |
| BPDU Guard | T1200 — Rogue switch / STP manipulation | Initial access | Yes (blocks); No (detection gap) |
| Inter-VLAN ACLs (both directions) | T1021 — Remote services (lateral movement) | Lateral movement | Yes |
| ASA zone enforcement + DMZ_IN ACL | T1210 — Exploitation of remote services | Lateral movement | Yes |
| OUTSIDE_IN ACL (host-specific, port-specific) | T1190 — Exploit public-facing application | Initial access | Partial (surface limited; servers unhardened) |
| SSH + VTY ACL 90 (IT only, core switches) | T1110 — Brute force; T1078 — Valid accounts | Credential access | Yes (core); No (access switches) |
| Privilege level separation (admin/support) | T1078.003 — Local accounts | Privilege escalation | Yes (core); No (access switches) |
| VLAN segmentation | T1021 — Remote services | Lateral movement | Yes |
| ASA NAT/PAT | T1590 — Network topology discovery (hides internal addressing) | Reconnaissance | Partial |
| — | T1557 — OSPF LSA injection | Adversary-in-the-middle | **No — not mitigated** |
| — | T1557 — HSRP gateway hijacking | Adversary-in-the-middle | Partial (ACL limits scope) |
| — | T1590 — CDP reconnaissance | Reconnaissance | **No — not mitigated** |

---

## NIST CSF 2.0 gap analysis

MITRE ATT&CK shows which attack techniques are mitigated. The NIST CSF 2.0 asks a different question: across the full security lifecycle, which stages does this network actually cover?

The honest finding, stated up front: the design is Protect-heavy and Detect/Respond/Recover-absent. That is a normal and defensible profile for a campus network lab focused on segmentation and access control, but it is a real gap and is named here explicitly rather than left for an auditor to find.

### GV — Govern

| Category | Implemented? | Notes |
| --- | --- | --- |
| GV.OC — Organisational context | Partial | Org profile and data sensitivity requirements documented in nexus-documentation.md. No formal risk tolerance statement. |
| GV.RM — Risk management strategy | No | No risk register, no risk acceptance criteria. Design implicitly treats HR/Finance as high-value but doesn't formalise this as a risk decision. |
| GV.RR — Roles and responsibilities | Partial | Admin/support privilege separation exists on core switches. No broader policy defining change authority, incident escalation, or ownership. |
| GV.PO — Policy | No | No formal security policy document. ACL and zone rationale in nexus-documentation.md functions as de facto policy but is not formalised. |
| GV.SC — Supply chain risk | N/A | Out of scope at this organisational scale. |

### ID — Identify

| Category | Implemented? | Notes |
| --- | --- | --- |
| ID.AM — Asset management | Yes | Physical inventory, IP/VLAN tables, and static assignments fully documented. |
| ID.RA — Risk assessment | Partial | MITRE ATT&CK mapping functions as a lightweight risk assessment. No likelihood/impact scoring or asset-level risk register. |
| ID.IM — Improvement | Partial | Challenges faced and open items sections function as an informal lessons-learned log. Not tied to a recurring review cadence. |

### PR — Protect (strongest area)

| Category | Implemented? | Notes |
| --- | --- | --- |
| PR.AA — Identity management and access control | Yes (core) / Partial (access) | `login local`, SSHv2, privilege levels, VTY ACL, port security on core switches. Access switches use shared Telnet passwords — closable through configuration, not a hardware limitation. |
| PR.DS — Data security | Partial | VLAN segmentation and ACLs protect data in transit between zones. No data-at-rest controls (out of scope for a Packet Tracer lab). |
| PR.PS — Platform security | Yes | DHCP snooping, DAI, port security, PortFast/BPDU Guard, scrypt enable secrets (core), ASA zone enforcement. Most of the project's engineering effort lives here. |
| PR.IR — Infrastructure resilience | Yes | HSRP, RPVST+, LACP EtherChannel, OSPF with floating static backup. Redundancy is well covered at L2/L3. |

### DE — Detect (most significant gap)

| Category | Implemented? | Notes |
| --- | --- | --- |
| DE.CM — Continuous monitoring | No | No syslog server, no SNMP, no NetFlow. Port security violations, BPDU Guard triggers, and DAI drops occur locally on individual devices and are never collected centrally. |
| DE.AE — Adverse event analysis | No | No log correlation or alerting. An ARP spoofing attempt that DAI correctly drops generates a local console message and nothing else. No one is notified unless physically watching that device's console. |

This is the clearest, most concrete gap in the design. The network can prevent and block a meaningful set of attacks, but there is no mechanism to know when those defences actually fire. A rogue device triggering BPDU Guard errdisables the port correctly, but nobody is told it happened. The cheapest first step into the Detect function is a syslog server in VLAN 30 with `logging host <address>` on every device — already listed as a planned addition in nexus-documentation.md.

### RS — Respond

| Category | Implemented? | Notes |
| --- | --- | --- |
| RS.MA — Incident management | No | No incident response plan. No documented process for what happens after, say, a BPDU Guard trigger or a DAI drop spike. Recovery is manual and undocumented. |
| RS.AN — Incident analysis | No | No forensic logging capability means there would be nothing to analyse even if an incident were noticed. |
| RS.CO — Incident communication | No | No defined escalation path or stakeholder communication plan. |

### RC — Recover

| Category | Implemented? | Notes |
| --- | --- | --- |
| RC.RP — Recovery planning | No | No configuration backup process. No disaster recovery plan. No defined RTO/RPO. If a core switch config were lost, recovery means manually re-entering everything from the documentation. |
| RC.CO — Recovery communication | No | Follows from RS.CO. No recovery communication plan exists. |

### Summary

| CSF Function | Coverage | Assessment |
| --- | --- | --- |
| Govern | Mostly absent | Informal only — rationale exists but isn't formalised as policy |
| Identify | Good | Asset inventory and IP/VLAN documentation are thorough |
| Protect | Strong | DAI, DHCP snooping, BPDU Guard, ASA zones, ACLs, port security, SSH, privilege separation |
| Detect | Absent | No monitoring, logging, or alerting anywhere in the topology |
| Respond | Absent | No incident response process |
| Recover | Absent | No config backup or recovery plan |

---

## Prioritised remediation list

Ordered by risk reduction per effort:

| Priority | Item | Effort | Risk addressed |
|---|---|---|---|
| 1 | OSPF MD5 authentication on all OSPF devices | Low — config only | T1557 OSPF injection — currently unmitigated, High risk |
| 2 | Syslog server in VLAN 30, `logging host` on all devices | Low — one server, one command per device | Entire DE (Detect) function — currently absent |
| 3 | OSPF passive interfaces on all access-facing SVIs | Low — config only | Prevents end devices from participating in OSPF |
| 4 | SSH on access switches (A-SW1 to A-SW4) | Low — config only | T1110 / T1078 on access layer — currently unmitigated |
| 5 | HSRP MD5 authentication on both core switches | Low — config only | T1557 HSRP hijack — currently partial |
| 6 | `no cdp enable` on access-facing ports | Low — config only | T1590 CDP reconnaissance — currently unmitigated |
| 7 | Fix floating static AD to 254 on both core switches | Low — one command change | Routing correctness; OSPF fallback as designed |
| 8 | Remove DMZ subnet from OSPF on ASA | Low — remove one network statement | Prevents internal route learning to DMZ; reinforces zone policy |
| 9 | Create loopback interfaces on C-SW1 and C-SW2 | Low — config only | Stable OSPF router ID; reachable SSH management target |
| 10 | Extended VTY ACL specifying destination and port 22 | Low-medium | Tightens SSH access to specific management addresses only |
