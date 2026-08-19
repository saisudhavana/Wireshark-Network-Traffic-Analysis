#  Data Exfiltration Case Study: Wireshark Network Forensics

This project documents the core concepts, behavioral indicators, and packet-level forensic investigation of a high-volume **Data Exfiltration over SSL/HTTPS** network event.

---

## 📖 Part 1: Core Cyber Security Concepts (Theory)

### 💡 1. What is Data Exfiltration?
**Data exfiltration** is the unauthorized transfer of data from an internal environment to an external or attacker-controlled destination.

Attackers may target:
* Credentials and authentication data
* Personal / customer information (PII)
* Confidential corporate documents
* Source code and database records
* Proprietary intellectual property

A typical attack flow follows this path:
```text
Compromise ──> Discovery ──> Collection ──> Staging ──> Exfiltration ──> Attacker C2 Server
```
*   **Collection** means gathering the target data inside the network.
*   **Exfiltration** means moving that gathered data outside the secure perimeter.

---

### 📥 2. Data Staging
Before stealing the assets, attackers usually stage the collected files in a hidden, temporary directory to organize the data pipeline:

```text
Sensitive Files ──> Temporary Directory ──> Compressed/Archived Data ──> External Destination
```
Compression (like `.zip`, `.tar.gz`, or `.7z`) or encryption is heavily used during staging to reduce transfer sizes and hide the file contents from network inspection scanners.

---

### 🌐 3. Common Exfiltration Methods

#### HTTP / HTTPS
Attackers transfer data through standard web traffic to an external server. HTTPS encrypts the contents, but network metadata such as IPs, ports, timing, and traffic volume spikes remain visible during monitoring.
```text
Internal Host ───> HTTP/HTTPS (Port 443) ───> External Attacker Server
```

#### Web Services / Cloud Storage
Attackers regularly abuse legitimate cloud services (like AWS S3 buckets, Google Drive, or OneDrive). This is difficult to detect because the destination URL itself belongs to a trusted platform provider.

#### DNS Exfiltration
Data can be encoded into pieces and hidden inside outgoing DNS query subdomains. Common indicators include unusually long domain strings, random-looking subdomains, high query frequencies, and thousands of unique requests.
```text
[Data Chunk 1].attacker.com ───> [Data Chunk 2].attacker.com ───> Rogue Nameserver
```

#### ICMP Covert Channels
ICMP can be abused to carry encoded data inside the optional payload field of ping packets instead of being used only for normal network routing diagnostics.

#### SSH / File Transfer
SSH-based protocols like SCP or SFTP are used to upload files to an external infrastructure. An unexpected, high-volume outbound connection over **TCP Port 22** is a strong indicator worth investigating.

---

### 🚨 4. Network Indicators
During structural traffic analysis, common indicators of possible data theft include:
* An internal host communicating with an unusual, previously unseen external IP address.
* Large or sudden unexpected outbound data volume spikes.
* Unusual destination ports or protocols leaving production networks.
* Continuous, repeated connections targeting the exact same external destination.
* A heavily skewed, high outbound-to-inbound (Upload vs. Download) traffic ratio.

> ⚠️ **Important:** None of these indicators alone proves data exfiltration. Context and baseline comparison are required. For example, a large upload could simply be an approved corporate cloud backup.

---

### 📊 5. Packet Size vs. Total Data
A common mistake during network analysis is confusing **packet length** with the **total data transferred**.

If Wireshark logs display a continuous sequence:
```text
Packet 1 ──> 1454 bytes
Packet 2 ──> 1454 bytes
Packet 3 ──> 1454 bytes
```
`1454 bytes` is simply the size of an *individual packet payload frame*. The actual volume of stolen data depends on the **sum of all packets combined**. Investigators must consider this formula:

```text
Packet Count + Packet Size + Stream Duration + Traffic Direction = Total Transfer Volume Pattern
```

---

### ⏱️ 6. Why Packet Timing Matters
Repeated connections or packet arrivals at perfectly uniform intervals indicate automated communication loops rather than human activity:

```text
Host ───> External IP [Fires continuously every few milliseconds / seconds]
```
However, regular timing alone is **not malicious** (e.g., normal system NTP updates or logging syncs). Timing becomes a useful forensic finding only when combined with an unusual destination IP, high upload volumes, and suspicious background endpoint processes.

---

## 🔬 Part 2: Technical Wireshark Investigation

### 🛠️ 1. Wireshark Investigation Focus Matrix
When trailing a suspected data exfiltration event, the packet-level investigation focuses on extracting these values:

| Area | Forensic Triage Objective |
| :--- | :--- |
| **Source IP** | Identify which internal workstation is sending data. |
| **Destination IP** | Map exactly where the data is going on the internet. |
| **Protocol / Port** | Determine how the data is moving (e.g., SSL on Port 443). |
| **Packet Length** | Measure the individual frame payload size. |
| **Packet Count** | Calculate the total data volume generated by the transfer. |
| **Time / Duration** | Isolate when the leak started and how long the channel stayed open. |

Every forensic capture file investigation must answer these fundamental core questions:
```text
WHO?        --> Which internal host is communicating?
WHERE?      --> Which external destination is receiving the traffic?
HOW?        --> Which protocol and port are being used to transport the data?
HOW MUCH?   --> How much total data was successfully transferred out?
WHEN?       --> When did the communication occur (office hours vs midnight)?
WHY?        --> Is there a legitimate enterprise explanation for this behavior?
```

---

### ⚖️ 2. Exfiltration vs. Legitimate Traffic Profiles

#### Potentially Legitimate Profile
```text
Internal Workstation ──> Approved Corporate Cloud Storage ──> Authorized Large File Upload
```

#### Potentially Suspicious Profile
```text
Internal Workstation ──> Unknown Script Process ──> Rare External IP ──> Massive Outbound Transfer
```
The strongest network evidence always comes from **multiple indicators occurring together**:
```text
Sensitive File Access + Data Staging/Compression + Unknown External IP + Large Outbound Transfer
```
## 3. MITRE ATT&CK

Data exfiltration falls under the **TA0010 — Exfiltration** tactic in MITRE ATT&CK.

Relevant techniques include:

* **T1041 — Exfiltration Over C2 Channel**
* **T1048 — Exfiltration Over Alternative Protocol**
* **T1567 — Exfiltration Over Web Service**
* **T1567.001 — Exfiltration to Code Repository**
* **T1567.002 — Exfiltration to Cloud Storage**

The technique depends on **how the attacker transfers the data**.

---

# 🔬 Part 2: Technical Wireshark Investigation

## 1. Environment Setup & Packet Acquisition

### Step 1 — Download the PCAP
Download the official data exfiltration practice capture file from the laboratory repository:
- [Direct Repository File: 05-data-exfiltration.pcap](https://github.com/geezsecurity/Pcap-Analyzer/blob/master/pcaps/05-data-exfiltration.pcap)
- [Direct Raw File Download Link](https://githubusercontent.com)

### Step 2 — Open the PCAP in Wireshark
Open the downloaded file `05-data-exfiltration.pcap` inside **Wireshark**.

---

## 2. Technical Investigation Findings & Suspicious Indicators

### Indicator 1 — Continuous High-Volume Stream from Source to Destination
#### Evidence
The packet list pane displays a heavy, aggressive, and unbroken sequence of outbound traffic originating from the internal network workstation (**`192.168.10.50`**) moving straight out to a single unknown internet endpoint IP address: **`89.248.165.30`**.

#### Forensic Meaning
Normal enterprise hosts engage in asymmetrical downloading patterns (small requests out, huge data payloads back). A continuous, monolithic stream flowing completely *outward* of the network boundary provides strong behavioral evidence of an active data extraction tunnel.

#### 📸 Wireshark Evidence: Outbound Upload Stream Monolith
![](/Attack-Scenarios/Data-Exfiltration/Screenshots/Data-Exfiltration-1.png)

---

### Indicator 2 — Repeated Packets with Identical Maximum Length (`1454 bytes`)
#### Evidence
The `Length` column shows that almost every single sequential packet line is locked exactly at a uniform maximum payload capacity of **`1454 bytes`**.

#### Forensic Meaning
Organic web browsing traffic creates variable packet sizes because humans look at distinct text sizes and loading images. A continuous block of identical maximum-capacity payloads proves a computer script is raw-streaming a large structured file (like a compressed `.zip` database dump) across the network interface card as fast as possible.

#### 📸 Wireshark Evidence: Monolithic Packet Payload Distribution
![](/Attack-Scenarios/Data-Exfiltration/Screenshots/Data-Exfiltration-2.png)

---

### Indicator 3 — Automated Sub-Millisecond Time Intervals
#### Evidence
Looking closely at the `Time` column, the timestamps reveal that packets are hitting the wire at an impossibly high speed, bursting exactly every **0.0001 to 0.0008 seconds** without a single pause.

#### Forensic Meaning
Humans operate with erratic delays and page-scrolling pauses. A perfectly tight, sub-millisecond arrival rate mathematically proves a high-speed programmatic loop utility is maximizing its upload bandwidth to drain data before a security alert can trip.

#### 📸 Wireshark Evidence: 
![](/Attack-Scenarios/Data-Exfiltration/Screenshots/Data-Exfiltration-1.png)

---

### Indicator 4 — Endless SSL "Continuation Data" Loop
#### Evidence
Immediately after the initial handshake finishes, the protocol flips entirely to **SSL**. The `Info` column fills up with an absolute block of continuous **`Continuation Data`** messages moving from left to right.

#### Forensic Meaning
This indicates that the workstation is dumping a massive file stream to that destination IP address without stopping to make any new web requests. The attacker is using the SSL encrypted tunnel to hide the stolen contents of the file, preventing the perimeter firewalls from reading the stolen databases.

#### 📸 Wireshark Evidence: 
![](/Attack-Scenarios/Data-Exfiltration/Screenshots/data-Exfiltration-3.png)

---

## 3. Wireshark Investigation Summary

| Indicator | Evidence Extracted from PCAP | Simple Forensic Meaning |
| :--- | :--- | :--- |
| **Outbound Upload Focus** | `192.168.10.50` $\rightarrow$ `89.248.165.30` | A single internal workstation is dumping bulk data to an external repository. |
| **Encrypted Tunneling** | Port `443` / SSL Protocol | The attacker is using encryption to hide the stolen file contents from perimeter firewalls. |
| **Monolithic Data Blocks** | Continuous `1454` byte packet blocks | Indicates a massive structured file is being split and streamed rapidly. |
| **Microsecond Pacing** | Packet intervals under `0.0008s` | Proves automated tool generation over organic human network usage. |


