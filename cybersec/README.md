# Cybersecurity & Networking Master Guides

> **Zero to world-class** curriculum for **big tech**, **government**, and **enterprise** security & networking roles.  
> Same format as the [system design guides](../README.md): parts, tables, worked examples, case studies, mermaid diagrams, cheat sheets, 24-month mastery paths.

---

## Guides

| Guide | File | Lines | Parts | Covers |
|-------|------|-------|-------|--------|
| **Cybersecurity Master** | [CYBERSEC-MASTER-GUIDE.md](CYBERSEC-MASTER-GUIDE.md) | ~7,957 | 0–20 | Linux, Wireshark, pentest, blue team, AD/Kerberos, threat hunting, malware forensics, DevSecOps, zero trust, multi-cloud/K8s, NIST/FedRAMP, world-class operator |
| **Unleash Hacking (Ethical)** | [UNLEASH-HACKING.md](UNLEASH-HACKING.md) | ~2,812 | 0–16 | OS mastery, **16 Python hack scripts**, nmap/Burp/Metasploit/BloodHound/hashcat, AD attacks (authorized), HTB→OSCP hero path |
| **Networking Master** | [NETWORKING-MASTER-GUIDE.md](NETWORKING-MASTER-GUIDE.md) | ~4,875 | 0–16 | Subnetting, routing, AWS/Azure/GCP VPC, network security, wireless, BGP, QoS/MTU, elite troubleshooting |

**Total:** ~15,644 lines across all guides.

---

## World-Class 24-Month Path (Both Guides)

### Quarters 1–2: Foundations (Never skip)

| Month | Cybersec | Networking |
|-------|----------|------------|
| 1 | Part 0–1 (CIA, legal, ethics) | Part 1 (subnetting until automatic) |
| 2 | Part 2–3 (Linux, lab, **Wireshark**) | Part 2–4 (OSI, VLANs, routing) |
| 3 | Part 4–5 (network security lens, crypto) | Part 5–6 (TCP/DNS, troubleshooting) |
| 4 | Part 6–7 (OWASP, Bash/Python scripting) | Part 8 (AWS VPC — all case studies) |

### Quarters 3–4: Operator (Offense + Defense)

| Month | Cybersec | Networking |
|-------|----------|------------|
| 5 | Part 8 (pentest, 3 case studies) | Part 12 (firewalls, IDS/IPS, DDoS) |
| 6 | Part 9 (SOC, SIEM hunts) | Part 13 (WiFi, VPN, ZTNA) |
| 7 | Part 10 (AWS security) | Part 14 (Azure + GCP networking) |
| 8 | Part 14–15 (AD security, threat hunting) | Part 15 (BGP, internet routing) |

### Quarters 5–6: World-Class (Breadth + Communication)

| Month | Cybersec | Networking |
|-------|----------|------------|
| 9 | Part 16 (malware / Volatility basics) | Part 16 (QoS, MTU/MSS, war stories) |
| 10 | Part 17 (DevSecOps, supply chain) | Re-read Part 8 + build hybrid cloud lab |
| 11 | Part 18–19 (zero trust, K8s security) | Cert prep (CCNA / AWS Advanced Networking) |
| 12 | Part 11 + 20 (gov compliance OR exec skills) | Part 10 + portfolio project |

**Certs to target:** Security+ → OSCP or CySA+ → CISSP / AWS Security / GIAC specialty.

---

## Cybersecurity Master — All 21 Parts (0–20)

See also: **[UNLEASH-HACKING.md](UNLEASH-HACKING.md)** — dedicated offensive track (Python scripts, tools, HTB→OSCP hero path).

| Part | Topic |
|------|-------|
| 0 | 12-month + **24-month world-class paths** |
| 1 | CIA triad, kill chain, MITRE ATT&CK, legal/ethical boundaries |
| 2 | **All Linux commands** for security engineers + cheat sheet |
| 3 | Lab setup, SSH/sudo/fail2ban, Burp, **Wireshark complete guide** (5 case studies), Metasploit layout |
| 4 | Networking essentials for security |
| 5 | Cryptography, TLS, OpenSSL |
| 6 | OWASP Top 10 + Burp workflow |
| 7 | Bash + Python security scripting |
| 8 | Ethical hacking / PTES + 3 pentest case studies |
| 9 | Blue team, SIEM, incident response, forensics |
| 10 | AWS cloud security (IAM, S3, GuardDuty, K8s intro) |
| 11 | Government: NIST 800-53, FedRAMP, STIGs, clearance |
| 12 | Big tech career, certs, interviews |
| 13 | Master cheat sheets + glossary |
| **14** | **Windows & Active Directory** — Kerberos, BloodHound, lateral movement, Event IDs |
| **15** | **Threat hunting & intel** — hunt loop, STIX/TAXII, Sigma/YARA, 5 hunt queries |
| **16** | **Malware analysis & memory forensics** — static/dynamic, Volatility, safe lab |
| **17** | **DevSecOps & supply chain** — SAST/DAST/SCA, CI/CD, SBOM, SLSA |
| **18** | **Zero trust architecture** — NIST 800-207, ZTNA, microsegmentation |
| **19** | **Multi-cloud & K8s security** — Azure/GCP, RBAC, NetworkPolicy |
| **20** | **World-class operator** — purple team, tabletops, exec briefings, 20 portfolio projects |

---

## Networking Master — All 17 Parts (0–16)

| Part | Topic |
|------|-------|
| 0 | Study paths + **24-month world-class network engineer path** |
| 1 | **Never Forget Subnetting** — apartment analogy, 15+ examples, VLSM |
| 2–5 | OSI, switching/VLANs, routing/NAT, TCP/DNS/DHCP/TLS |
| 6 | Troubleshooting toolkit + 7 case studies |
| 7 | WAN, VPN, load balancing |
| 8 | **AWS networking** — VPC, IGW, NAT, SG vs NACL, TGW, Route 53, 4 architecture cases |
| 9 | Spine-leaf, VXLAN, Terraform VPC |
| 10–11 | Cert path, cheat sheets |
| **12** | **Network security** — firewalls, Snort/Suricata, segmentation, DDoS, WAF |
| **13** | **Wireless & remote access** — WPA3, 802.1X, IPSec, ZTNA vs VPN |
| **14** | **Azure & GCP networking** — VNet, Cloud Armor, hybrid multi-cloud cases |
| **15** | **BGP & internet routing** — AS, hijacking, anycast, CDN |
| **16** | **Performance & elite troubleshooting** — MTU/MSS black hole, QoS, 5 war stories |

---

## Lab Prerequisites

| Tool | Purpose |
|------|---------|
| VirtualBox / VMware | Isolated lab (Kali + Ubuntu + Windows Server AD) |
| AWS / Azure free tier | Cloud networking + security labs |
| Wireshark + Burp Suite | Traffic analysis + web testing |
| BloodHound + Impacket | AD lab only (authorized) |
| Volatility + REMnux | Malware analysis lab (isolated network) |

**Warning:** Only test systems you own or have **written authorization** to test.

---

## Cross-References

| Related | Location |
|---------|----------|
| System design networking | [27-networking-for-system-design.md](../08-fundamentals/27-networking-for-system-design.md) |
| DNS deep dive | [28-dns-deep-dive.md](../08-fundamentals/28-dns-deep-dive.md) |
| AWS service mapping | [31-cloud-infrastructure-service-mapping.md](../09-infrastructure/31-cloud-infrastructure-service-mapping.md) |
| API Gateway / mTLS | [34-api-gateway-service-mesh.md](../09-infrastructure/34-api-gateway-service-mesh.md) |
| Capacity estimation | [CAPACITY-ESTIMATION-MASTER-GUIDE.md](../CAPACITY-ESTIMATION-MASTER-GUIDE.md) |

---

## What "World-Class" Means Here

| Skill | How this curriculum builds it |
|-------|------------------------------|
| **Deep technical** | Linux, Wireshark, AD, cloud, K8s, subnetting, BGP |
| **Offensive mindset** | Pentest methodology + attacker TTPs (authorized labs) |
| **Defensive operations** | SOC, hunting, IR, forensics, zero trust |
| **Engineering** | Scripting, DevSecOps, automation, multi-cloud |
| **Governance** | NIST, FedRAMP, risk, compliance for gov roles |
| **Communication** | Reports, exec briefings, tabletops (Part 20) |

World-class is not reading once — it is **labs, reports, certs, and portfolio projects** over 24 months.

---

*Last updated: July 2026*
