# Wireshark-Network-Traffic-Analysis
Hands-on Wireshark network traffic analysis, protocol investigation, and attack detection using PCAPs, MITRE ATT&amp;CK, and KQL.

# Wireshark Network Traffic Analysis & Threat Investigation

## Overview

This repository documents my hands-on learning and investigation of
network traffic using Wireshark.

The project is divided into two sections:

1. Protocol Analysis
2. Attack Scenario Investigation

The first section focuses on understanding normal network protocols
and identifying how they appear in packet captures.

The second section applies this knowledge to investigate suspicious
network activity and security scenarios.

---

## Objectives

- Understand common network protocols at packet level
- Analyze network traffic using Wireshark
- Identify normal vs suspicious traffic patterns
- Investigate real-world attack scenarios using PCAP files
- Develop network threat-hunting skills
- Map relevant attacks to MITRE ATT&CK
- Recreate detection logic using KQL where applicable
- Understand how Microsoft Defender and Microsoft Sentinel
  can detect and respond to network threats

---

# 01 — Protocol Analysis

The following protocols are analyzed at packet level:

| Protocol | Topics Covered |
|---|---|
| ARP | ARP Request, ARP Reply, MAC resolution |
| DNS | DNS Query, Response, Record Types |
| DHCP | Discover, Offer, Request, ACK |
| TCP | 3-Way Handshake, Flags, Ports |
| HTTP | GET, POST, HTTP headers |
| TLS | TLS Handshake and encrypted traffic |
| ICMP | Echo Request, Echo Reply |
| SMTP | Email communication and SMTP commands |

---

# 02 — Attack Scenario Investigation

The following security scenarios are investigated using
Wireshark and PCAP files:

- DNS Tunneling
- Port Scanning
- SYN Scanning
- SYN Flood
- ICMP Flood
- Data Exfiltration
- C2 Beaconing
- Man-in-the-Middle Attacks
- Malware Network Traffic
- Additional network-based attack scenarios

Each investigation focuses on:

1. Understanding the attack
2. Investigating the PCAP in Wireshark
3. Identifying suspicious network indicators
4. Collecting packet-level evidence
5. Mapping to MITRE ATT&CK where applicable
6. Recreating detection logic using KQL
7. Understanding MDE/Sentinel detection
8. Documenting remediation steps

---

# Tools

Wireshark → practical PCAP investigation ✅
KQL / Sentinel → detection logic and hunting query ✅
MITRE ATT&CK → mapping ✅

---

# Investigation Methodology

The general investigation process used throughout this project is:

PCAP
↓
Protocol Filtering
↓
Traffic Analysis
↓
Suspicious Indicators
↓
Packet-Level Evidence
↓
MITRE ATT&CK Mapping
↓
KQL Detection Logic
↓
Remediation

---

# Disclaimer

The PCAP files used in this repository are used for
educational and security-analysis purposes.
