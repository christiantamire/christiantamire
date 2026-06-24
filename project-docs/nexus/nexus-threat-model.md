> Fictional scenario. Nexus Consulting Group does not exist. This document was produced as part of a personal portfolio project.

Author: Musasizi Christian Samuel

Last updated: June 2026

Companion document: [nexus-documentation.md](./nexus-documentation.md)

---

## Purpose

This document answers the question the network documentation does not: is the Nexus Consulting Group network actually secure, and against what threats? Where `nexus-documentation.md` records what was built and how, this document records what the network is defending against, how each threat is analysed, how risk is scored, and what the outcome is in terms of residual exposure and remediation priority.

The analysis uses two frameworks together because they answer different questions. MITRE ATT&CK maps specific adversary techniques to specific controls. NIST CSF 2.0 maps the full security lifecycle. Both are used rather than one, since they answer different questions and neither alone gives a complete picture.

---

## 1. Scope

### Network boundary

This threat model covers the Nexus Consulting Group campus network as designed and simulated in Cisco Packet Tracer. The boundary runs from the ISP handoff at Router0's Fa0/0 interface to the endpoint access ports on A-SW1 through A-SW4. Everything inside that boundary is in scope. The ISP infrastructure, endpoint operating systems, and application-layer software running on DMZ servers are out of scope.

### Assets in scope

| Asset | Classification | Rationale |
| --- | --- | --- |
| Core switches C-SW1, C-SW2 | Critical | Compromise gives an attacker control over routing, ACLs, and VLAN policy for the entire network |
| ASA firewall (ASA0) | Critical | Sole enforcement point between internal zones and the internet; bypass eliminates perimeter controls entirely |
| Finance VLAN (VLAN 50) | High | Payroll data, accounts, regulatory filings; exposure creates direct financial and legal liability |
| HR VLAN (VLAN 20) | High | Personal staff records, recruitment data; subject to data protection obligations |
| SERVERS VLAN (VLAN 30) | Medium-High | DHCP and DNS infrastructure; compromise enables redirection attacks across all VLANs |
| IT VLAN (VLAN 10) | Medium | Administrative access subnet; compromise gives attacker SSH access to network infrastructure |
| DMZ servers (web, FTP, DNS) | Medium | Internet-facing; compromise could pivot inward or damage firm reputation |
| Access switches A-SW1 to A-SW4 | Medium | Management plane is weakly protected; see Scenario 10 |

### Threat actors in scope

**External opportunistic attacker** — automated scanning and exploitation of internet-facing services. Targets DMZ servers and any exposed management ports. Low sophistication, high volume. Motivation: data theft, ransomware, botnet recruitment.

**Targeted external attacker** — deliberate reconnaissance against client data or financial records. Higher sophistication, willing to chain multiple techniques. Motivation: competitive intelligence, financial fraud, extortion.

**Malicious or negligent insider** — a staff member who intentionally or accidentally accesses systems beyond their authorisation. Threat is horizontal (lateral movement between departments) rather than perimeter-focused.

No nation-state or APT threat is assumed at this organisational scale.

### Frameworks used

| Framework | Version | Purpose in this document |
| --- | --- | --- |
| MITRE ATT&CK | Enterprise v14 | Maps adversary techniques to controls per attack scenario |
| NIST CSF | 2.0 | Assesses security lifecycle coverage across Govern, Identify, Protect, Detect, Respond, Recover |

### Out of scope

- ISP infrastructure and upstream routing
- Endpoint OS hardening and patch management
- Application-layer security of DMZ server software
- Data-at-rest encryption on endpoints or servers
- Physical security of the office premises

---

## 2. Threat Analysis

Each scenario follows the same structure: attack description, MITRE technique, affected assets, control implemented, and an initial residual risk statement before scoring.

---

### Scenario 1 — Rogue DHCP server

An attacker plugs a device into an access port and runs a DHCP server, handing out a fake default gateway or DNS server to clients on the same VLAN. Clients that accept the fake lease route all traffic through the attacker.

| | |
|---|---|
| MITRE technique | T1557.003 — DHCP spoofing |
| Tactic | Credential access / adversary-in-the-middle |
| Threat actor | Insider, compromised endpoint |
| Affected assets | Any VLAN where the rogue device gains access |
| Control implemented | DHCP snooping on all access switches. All access ports untrusted by default; uplinks toward core switches are trusted. DHCP server responses on untrusted ports are dropped before reaching clients. |
| Residual risk statement | Low. DHCP snooping directly and completely addresses this technique. |

---

### Scenario 2 — ARP cache poisoning

An attacker on the same VLAN broadcasts gratuitous ARP replies claiming ownership of another device's IP (e.g., the default gateway). Clients update their ARP tables and send traffic to the attacker instead.

| | |
|---|---|
| MITRE technique | T1557.002 — ARP cache poisoning |
| Tactic | Credential access / adversary-in-the-middle |
| Threat actor | Insider, compromised endpoint |
| Affected assets | Any VLAN the attacker has access to |
| Control implemented | Dynamic ARP Inspection (DAI) validates every ARP reply against the DHCP snooping binding table using `validate dst-mac ip src-mac` checks. A reply claiming an IP not matching the binding is dropped. |
| Residual risk statement | Low for DHCP clients. Static IP devices (VLAN 30 servers, printer) are not covered by the binding table; Packet Tracer does not support ARP ACLs for static entries so those ports rely on interface-level trust. Platform limitation, not a design choice. |

---

### Scenario 3 — STP root bridge takeover

An attacker connects an unauthorised switch to an access port and sends superior BPDUs, causing STP to elect their device as root bridge. This repositions the attacker's switch in the forwarding path for some or all VLANs, enabling traffic interception.

| | |
|---|---|
| MITRE technique | T1200 — Hardware additions (rogue switch) |
| Tactic | Initial access / adversary-in-the-middle |
| Threat actor | Insider, physical access |
| Affected assets | All VLANs; STP topology is network-wide |
| Control implemented | BPDU Guard on all access ports errdisables the port the moment any BPDU arrives. Port security (sticky MAC, max 2, violation shutdown) provides a second layer: an unauthorised device triggers a violation before it can participate in STP. |
| Residual risk statement | Low for blocking. Gap exists in detection: the port errdisables correctly but no centralised alert fires. |

---

### Scenario 4 — MAC flooding (CAM table exhaustion)

An attacker floods the switch with frames using random source MAC addresses, filling the CAM table. Once full, the switch falls back to flooding all frames out all ports in the VLAN, allowing the attacker to capture traffic not addressed to them.

| | |
|---|---|
| MITRE technique | T1200 — Hardware additions; T1557 via flooding |
| Tactic | Collection / credential access |
| Threat actor | Insider, compromised endpoint |
| Affected assets | Any VLAN the attacker has physical access to |
| Control implemented | Port security (sticky MAC) limits each access port to 2 MAC addresses. A third MAC triggers errdisabled immediately, stopping the flood before the CAM table can be exhausted. |
| Residual risk statement | Low. |

---

### Scenario 5 — OSPF LSA injection

An attacker on the network segment crafts OSPF Hello packets to form an unauthorised adjacency with a core switch, then injects LSAs advertising more-specific prefixes (e.g. a /29 within a legitimate /26). The longest-prefix match rule causes routers to forward traffic for those hosts toward the attacker rather than the legitimate destination — a silent man-in-the-middle at the routing layer.

This attack is well-documented. Researcher Gabi Nakibly demonstrated persistent LSA injection at Black Hat 2011 against Cisco IOS 15.x, showing that uncompromised routers can be tricked into propagating false LSAs without triggering OSPF's fight-back mechanism.

| | |
|---|---|
| MITRE technique | T1557 — Adversary-in-the-middle; T1599 — Network boundary bridging |
| Tactic | Adversary-in-the-middle / collection |
| Threat actor | Targeted external attacker, insider |
| Affected assets | Routing table integrity; all traffic transiting core switches |
| Control implemented | **None.** OSPF is running without MD5 authentication. Any device that can reach 224.0.0.5 (AllSPFRouters multicast) on the segment can attempt adjacency formation. |
| Residual risk statement | High. Most significant unmitigated routing-plane risk in the current design. |

---

### Scenario 6 — HSRP gateway hijacking

An attacker crafts HSRP Hello messages with a higher priority than the legitimate active router, causing HSRP to elect the attacker's device as the active gateway. All traffic from the affected VLAN is then forwarded through the attacker.

| | |
|---|---|
| MITRE technique | T1557 — Adversary-in-the-middle |
| Tactic | Adversary-in-the-middle |
| Threat actor | Insider, compromised IT endpoint |
| Affected assets | All traffic from the hijacked VLAN's hosts |
| Control implemented | Partial. Inter-VLAN ACLs prevent non-IT VLANs from reaching SVI and HSRP IPs, limiting the attack surface. HSRP MD5 authentication is not configured. |
| Residual risk statement | Medium. Non-IT VLANs are partially protected by ACL policy, but a compromised IT endpoint or device on the same segment as the HSRP speakers could still attempt this. |

---

### Scenario 7 — Lateral movement between departments

A Finance staff member attempts to access HR systems, or a compromised Sales endpoint attempts to pivot to Finance. Cross-department traffic is the primary insider threat vector for a consulting firm with sensitive HR and Finance data.

| | |
|---|---|
| MITRE technique | T1021 — Remote services |
| Tactic | Lateral movement |
| Threat actor | Insider, compromised endpoint |
| Affected assets | HR VLAN (VLAN 20), Finance VLAN (VLAN 50) |
| Control implemented | Inter-VLAN ACLs on both core switches in both directions. Finance cannot initiate traffic to HR, Sales, or any other department. ACLs are applied inbound (from source VLAN) and outbound (toward destination VLAN), providing two enforcement points regardless of which core switch is active. |
| Residual risk statement | Low for network-layer lateral movement. No east-west detection capability exists for unusual traffic patterns within permitted flows. |

---

### Scenario 8 — Exploitation of public-facing DMZ services

An external attacker targets the web server (80/443), FTP server (20/21), or DNS server (53) from the internet, seeking to exploit a vulnerability in the service software.

| | |
|---|---|
| MITRE technique | T1190 — Exploit public-facing application |
| Tactic | Initial access |
| Threat actor | External opportunistic, targeted external |
| Affected assets | DMZ web, FTP, DNS servers |
| Control implemented | `OUTSIDE_IN` ACL on the ASA permits only specific ports to specific DMZ server IPs. All other inbound traffic from OUTSIDE is denied. The ACL is host-specific: web traffic only reaches 20.0.0.18, FTP only reaches 20.0.0.12, DNS only reaches 20.0.0.5. |
| Residual risk statement | Medium. ACL policy limits exposed surface area, but DMZ servers are Packet Tracer placeholders — no OS hardening, no WAF, no IPS in front of them. |

---

### Scenario 9 — DMZ pivot to internal network

An attacker who compromises a DMZ server attempts to use it as a pivot point to attack internal hosts.

| | |
|---|---|
| MITRE technique | T1210 — Exploitation of remote services |
| Tactic | Lateral movement |
| Threat actor | Targeted external attacker |
| Affected assets | Internal VLANs, core switches |
| Control implemented | `DMZ_IN` ACL blocks DMZ servers from initiating connections to internal hosts. ASA zone model (DMZ security level 50, INTERNAL security level 100) enforces this at the firewall level. A compromised DMZ server cannot reach 192.168.x.x addresses. |
| Residual risk statement | Low for direct connection attempts. Note: the DMZ subnet is inadvertently advertised into OSPF (see open items in nexus-documentation.md), which should be corrected to fully reinforce zone enforcement. |

---

### Scenario 10 — Brute force of management interfaces

An attacker with network access attempts to brute-force SSH credentials on core switches, or Telnet credentials on access switches.

| | |
|---|---|
| MITRE technique | T1110 — Brute force; T1078 — Valid accounts |
| Tactic | Credential access / privilege escalation |
| Threat actor | Insider, compromised endpoint |
| Affected assets | Core switches (C-SW1, C-SW2), access switches (A-SW1 to A-SW4) |
| Control implemented (core) | VTY ACL 90 restricts SSH attempts to the IT subnet (192.168.1.0/24) only. Per-user `login local` authentication. SSHv2 with 2048-bit RSA. Scrypt-hashed enable secret. |
| Control implemented (access) | None meaningful. Telnet with a shared line password. No `access-class` restricting source subnets. Any device that can route to an access switch management IP can attempt the shared password. |
| Residual risk statement | Low for core switches. High for access switches. |

---

### Scenario 11 — Privilege escalation via shared credentials

An attacker who obtains the shared access-switch line password gains full device access with no individual accountability.

| | |
|---|---|
| MITRE technique | T1078.003 — Local accounts |
| Tactic | Privilege escalation |
| Threat actor | Insider |
| Affected assets | Access switches A-SW1 to A-SW4 |
| Control implemented | Core switches use per-user `login local` with privilege level separation (admin/support). Access switches use shared passwords only — no per-user accounts, no privilege separation. |
| Residual risk statement | Medium. Fully mitigated on core switches; not mitigated on access switches. |

---

### Scenario 12 — CDP topology reconnaissance

An attacker connected to any access port passively receives CDP advertisements from the connected switch, revealing device model, IOS version, IP addresses, and native VLAN. This is free intelligence for planning further attacks without generating any traffic.

| | |
|---|---|
| MITRE technique | T1590 — Gather victim network information |
| Tactic | Reconnaissance |
| Threat actor | Any attacker with physical access to an access port |
| Affected assets | Network topology confidentiality |
| Control implemented | Addressed on access switches. `no cdp run` disables CDP globally on each access switch, then `cdp enable` re-enables it only on the trusted uplinks (g0/1-2) toward the core switches. Access-facing ports (f0/1-4) run no CDP at all. Core switches still run CDP on all interfaces by default, including the trunk ports facing access switches (g1/0/2-5) — not reachable from end-host ports, but visible to anyone who gains a foothold on the trunk segment itself (e.g. via Scenario 3). |
| Residual risk statement | Low. The original gap — CDP visible from any end-host access port — is closed. Remaining exposure is limited to the access-switch uplink trunks and core-switch trunk interfaces, which require already having a position on the inter-switch segment rather than a standard user port. |

---

## 3. Risk Scoring

Each scenario is scored using a 5×5 likelihood × impact matrix. Scores are assigned based on attacker access requirements, control effectiveness, and asset sensitivity.

**Scoring key**

| Level | Likelihood | Impact |
|---|---|---|
| 1 | Very unlikely — requires privileged access or rare conditions | Negligible — no data exposure, no service disruption |
| 2 | Unlikely — requires physical access or existing foothold | Minor — limited scope, quickly recoverable |
| 3 | Possible — accessible to any insider or compromised endpoint | Moderate — department-level exposure or partial service disruption |
| 4 | Likely — low barrier, standard tools sufficient | Significant — multi-department exposure or critical service disruption |
| 5 | Almost certain — no controls, trivially exploitable | Severe — full network compromise, data exfiltration, or prolonged outage |

**Risk score = Likelihood × Impact. Inherent = before controls. Residual = after controls.**

| # | Scenario | Inherent L | Inherent I | Inherent Score | Residual L | Residual I | Residual Score | Rating |
|---|---|---|---|---|---|---|---|---|
| 1 | Rogue DHCP server | 4 | 4 | 16 | 1 | 2 | 2 | Low |
| 2 | ARP cache poisoning | 4 | 4 | 16 | 1 | 2 | 2 | Low |
| 3 | STP root bridge takeover | 3 | 5 | 15 | 1 | 2 | 2 | Low |
| 4 | MAC flooding | 3 | 3 | 9 | 1 | 1 | 1 | Low |
| 5 | OSPF LSA injection | 3 | 5 | 15 | 3 | 5 | **15** | **High** |
| 6 | HSRP gateway hijacking | 3 | 4 | 12 | 2 | 4 | 8 | Medium |
| 7 | Lateral movement between departments | 4 | 4 | 16 | 1 | 2 | 2 | Low |
| 8 | Exploitation of DMZ services | 5 | 3 | 15 | 3 | 3 | 9 | Medium |
| 9 | DMZ pivot to internal | 3 | 5 | 15 | 1 | 3 | 3 | Low |
| 10 | Brute force of management interfaces | 4 | 5 | 20 | 1 (core) / 4 (access) | 3 (core) / 5 (access) | 3 (core) / **20 (access)** | Low / **High** |
| 11 | Privilege escalation via shared credentials | 3 | 4 | 12 | 1 (core) / 3 (access) | 2 (core) / 4 (access) | 2 (core) / **12 (access)** | Low / **High** |
| 12 | CDP topology reconnaissance | 5 | 2 | 10 | 1 | 1 | **1** | Low |

**Residual risk heat map**

```
Impact
  5 |        |        |   [5]  |        |        |
  4 |        |        |        |  [11a] |        |
  3 |        |        |  [8]   |        |        |
  2 | [1][2] |[3][4]  |  [9]   | [6]    |        |
  1 | [12]   |        |        |        |        |
    |   1    |   2    |   3    |    4   |    5   | Likelihood
    
[5] OSPF LSA injection — High
[10a] Access switch brute force — High  
[11a] Access switch privilege escalation — High
[6] HSRP hijacking — Medium
[8] DMZ exploitation — Medium
[12] CDP reconnaissance — Low
```

---

## 4. Visual Models

> *Placeholder — to be added in a subsequent revision. Planned visuals: attack surface diagram overlaid on logical topology, NIST CSF coverage wheel, and per-VLAN threat exposure map.*

---

## 5. Validation

> *Placeholder — to be completed once a companion GNS3/Eve-NG attack lab is built. Planned validation: Scapy-based OSPF adjacency test (unauthenticated vs MD5-authenticated), Yersinia STP root bridge attack, HSRP hijack attempt via Loki, Wireshark capture confirming DAI drops on ARP spoofing attempt.*

---

## 6. Outcome

### NIST CSF 2.0 lifecycle coverage

| CSF Function | Coverage | Assessment |
| --- | --- | --- |
| Govern | Mostly absent | Rationale exists informally; not formalised as policy or risk register |
| Identify | Good | Asset inventory, IP/VLAN planning, and this threat model's asset classification cover this well |
| Protect | Strong | DAI, DHCP snooping, BPDU Guard, ASA zones, inter-VLAN ACLs, port security, SSH, privilege separation, CDP scoped to trusted uplinks |
| Detect | Absent | No syslog, no SNMP, no NetFlow; control triggers (BPDU Guard, DAI, port security) are invisible beyond local console |
| Respond | Absent | No incident response plan, no escalation path, no forensic capability |
| Recover | Absent | No configuration backup process, no RTO/RPO, no recovery plan |

The design is Protect-heavy and Detect/Respond/Recover-absent. That is a defensible profile for a network scoped around segmentation and perimeter defense, but it means a successful attack would not be detected until damage was already visible. The cheapest immediate step into the Detect function is a syslog server in VLAN 30 with `logging host` on every device.

### MITRE ATT&CK coverage summary

| ATT&CK technique | Tactic | Mitigated? |
| --- | --- | --- |
| T1557.003 — DHCP spoofing | Credential access | Yes |
| T1557.002 — ARP cache poisoning | Credential access | Yes (DHCP clients); Partial (static IPs) |
| T1200 — Hardware additions (rogue switch, MAC flood) | Initial access | Yes |
| T1021 — Remote services (lateral movement) | Lateral movement | Yes |
| T1210 — Exploitation of remote services (DMZ pivot) | Lateral movement | Yes |
| T1190 — Exploit public-facing application | Initial access | Partial (surface limited; servers unhardened) |
| T1110 — Brute force | Credential access | Yes (core); **No (access switches)** |
| T1078 / T1078.003 — Valid accounts / local accounts | Privilege escalation | Yes (core); **No (access switches)** |
| T1557 — OSPF LSA injection | Adversary-in-the-middle | **No — not mitigated** |
| T1557 — HSRP gateway hijacking | Adversary-in-the-middle | Partial (ACL limits scope only) |
| T1590 — CDP reconnaissance | Reconnaissance | Yes (access-facing ports); Partial (core/access trunk interfaces still run CDP) |
| T1599 — Network boundary bridging | Adversary-in-the-middle | **No — not mitigated** |

### Prioritised remediation

Ordered by residual risk score, highest first.

| Priority | Residual Score | Item | Effort | Technique addressed |
|---|---|---|---|---|
| 1 | 15 | OSPF MD5 authentication on all OSPF devices | Low — config only | T1557 OSPF injection |
| 2 | 20 | SSH + per-user accounts on access switches (A-SW1 to A-SW4) | Low — config only | T1110, T1078 access layer |
| 3 | 12 | Privilege separation on access switches | Low — config only | T1078.003 access layer |
| 4 | 9 | DMZ server hardening (OS, WAF, IPS) | Medium — production scope | T1190 DMZ exploitation |
| 5 | 8 | HSRP MD5 authentication on both core switches | Low — config only | T1557 HSRP hijacking |
| 6 | — | Syslog server in VLAN 30, `logging host` on all devices | Low — one server | NIST DE (Detect) function entirely |
| 7 | — | Disable CDP on core-switch trunk interfaces facing access switches (g1/0/2-5), or restrict to need-only | Low — config only | Closes remaining T1590 exposure on inter-switch trunks |
| 8 | — | Fix floating static AD to 254 on both core switches | Low — one command | Routing correctness |
| 9 | — | Remove DMZ subnet from OSPF on ASA | Low — remove one network statement | Zone enforcement integrity |
| 10 | — | Create loopback interfaces on C-SW1 and C-SW2 | Low — config only | Stable router ID, SSH management target |
| 11 | — | Extended VTY ACL with destination and port 22 | Low-medium | Tightens SSH to specific management addresses |

**Note on the OSPF passive-interface remediation item from the prior revision:** `passive-interface` statements for gigabitEthernet 1/0/2–1/0/5 are present in the running configuration on both core switches, but they sit on interfaces configured as Layer 2 trunk switchports rather than Layer 3 SVIs — `passive-interface` has no effect on a switchport interface, since no OSPF process runs there in the first place. The statements are inert rather than functioning as a control. If the intent is to reduce the OSPF attack surface on VLANs that don't need to peer (e.g. VLAN 10, 20, 40, 50), `passive-interface` needs to be applied to those VLAN SVIs instead, leaving only VLAN 30 (where C-SW1 and C-SW2 peer with each other) and the link toward the firewall active. This remains an open item and is reflected in Scenario 5's "None" control status, since dead config does not reduce residual risk.
