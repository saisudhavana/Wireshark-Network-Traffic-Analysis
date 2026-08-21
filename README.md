# 🌐 Network Forensics & Threat Hunting Archive
### 🔬 Packet-Level Incident Response Case Studies with Wireshark

This repository serves as a dedicated network security portfolio documenting the forensic triage, packet-layer mechanics, and blue-team mitigation strategies for critical threat vectors. Every case study is backed by live packet capture (`.pcap`) deep-dives, protocol-layer analysis, and MITRE ATT&CK framework mapping.

---

## 🗺️ Master Security Scenario Index

| Project Directory | Core Protocol | Targeted Layer | MITRE ATT&CK ID | Primary Key Forensic Indicator |
| :--- | :--- | :--- | :--- | :--- |
| **[01-DNS-Tunneling](/Attack-Scenarios/DNS-Tuneling)** | DNS (UDP/53) | L7: Application | `T1071.004` | High-frequency TXT records with high-entropy subdomain strings. |
| **[02-Port-Scanning](/Attack-Scenarios/PortScanning)** | TCP | L4: Transport | `T1046` | High-velocity half-open `[SYN]` sweeps across multiple ports. |
| **[03-SYN-Flood](/Attack-Scenarios/Syn-Flood-Attack)** | TCP | L4: Transport | `T1498.001` | High-volume `[SYN]` blast targeting Port 25565 sharing a single Source MAC. |
| **[04-Data-Exfiltration](/Attack-Scenarios/Data-Exfiltration)** | SSL/TLS (HTTPS) | L7: Application | `T1567.002` | Monolithic outbound streams with locked `1454-byte` payload blocks. |
| **[05-ARP-Spoofing-MITM](/Attack-Scenarios/ARP-Spoofing)** | ARP | L2: Data-Link | `T1557.001` | Forged gateway IP replies mapping `192.168.1.1` to a workspace MAC. |

---

## 🔬 Core Wireshark Investigation Summaries

### 🛠️ 1. DNS Tunneling (Command & Control / Covert Channel)
*   **The Scenario:** An internal infected asset utilizes the Domain Name System protocol to bypass perimeter firewalls, establishing a hidden Command & Control (C2) communication channel to leak data out of the network block.
*   **Forensic Finding:** Identification of highly repetitive UDP Port 53 queries containing random-looking, long hexadecimal subdomains requesting `TXT` and `CNAME` records from an unapproved external authoritative name server.

### 🔍 2. Port Scanning (Reconnaissance Phase)
*   **The Scenario:** A local threat actor executes a rapid host and service discovery sweep across the internal network subnet to find active servers and software vulnerabilities.
*   **Forensic Finding:** Isolated host terminal firing hundreds of TCP `[SYN]` packets across sequential destination ports within milliseconds, monitoring for `[SYN, ACK]` or `[RST, ACK]` replies to map open sockets.

### 🌊 3. TCP SYN Flood Attack (Denial of Service)
*   **The Scenario:** An automated script targets a production gaming server (**Port 25565 - Minecraft**) to exhaust its connection memory space (**SYN Backlog Queue**), crashing the multiplayer application instance.
*   **Forensic Finding:** A sub-millisecond stream of thousands of fake `[SYN]` packets targeting a single port. While the digital Layer-3 IP addresses change constantly to mimic a DDoS, the Layer-2 physical **Source MAC Address stays perfectly identical**, mathematically proving an internal Spoofed DoS attack.

### 📦 4. Data Exfiltration (Encrypted Web Theft)
*   **The Scenario:** A compromised back-office machine consolidates customer database logs into a compressed staging file and dumps the data directly to an unapproved external cloud storage repository.
*   **Forensic Finding:** Analysis of an encrypted HTTPS/SSL pipeline over Port 443 showing a heavy upload-to-download imbalance. The automated script streams uninterrupted blocks of maximum transmission capacity (**locked payload lengths of `1454 bytes`**) at microsecond automated pacing.

### 🎭 5. ARP Spoofing (Adversary-in-the-Middle Routing Hijack)
*   **The Scenario:** An internal attacker machine intercepts the communication path between a target workstation (`192.168.1.104`) and the default network gateway router (`192.168.1.1`) to sniff data traffic frames in real time.
*   **Forensic Finding:** Dissection of forged unsolicited ARP replies where the attacker IP (`192.168.1.105`) writes the router's IP into the sender header but appends their own hardware MAC address card (`2d:f8:5a`). This creates a structural collision, triggering Wireshark’s internal expert warning: `[Duplicate IP address detected for 192.168.1.1]`.

---
*Portfolio maintained for network traffic analysis verification, incident triage tracking, and packet-level forensic validation.*
