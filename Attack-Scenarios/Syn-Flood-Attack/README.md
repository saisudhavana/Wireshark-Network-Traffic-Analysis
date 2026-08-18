# 🚪 TCP SYN Flood Attack Investigation Case Study

This case study documents the baseline theory, network-layer mechanics, and packet-level forensic investigation of a high-volume **TCP SYN Flood Attack** using Wireshark.

---

## 🛠️ Part 1: Core Cyber Security Concepts

### 💡 1. What is a TCP SYN Flood?

#### Technical Definition
A **TCP SYN Flood** is a protocol-level denial-of-service attack that targets the **TCP 3-Way Handshake** infrastructure. The attacker floods a target server with a continuous stream of initial connection requests (`SYN` packets), forcing the server to exhaust its **SYN Backlog Queue** (memory space reserved for half-open connections). This starves the system of available connection sockets, blocking legitimate traffic from connecting.

#### How a Normal TCP Connection Works
A normal TCP connection uses a structured 3-way handshake to establish communication:

```text
Client (e.g., Browser)                  Server (e.g., Flipkart)

   Step 1: [SYN] ------------------------> (Request to open shopping page)

                 <------------------  Step 2: [SYN, ACK] (Server reserves memory row)

   Step 3: [ACK] ------------------------> (Handshake finishes, loading cart)

                     [ Connection Established ]
```

#### What Happens During a SYN Flood?
During a SYN Flood, an attacker runs an automated script that bypasses Step 3 entirely. The script fires thousands of initial `[SYN]` packets per second directly at the server. 

```text
Attacker                               Server

   SYN  ------------------------> (Allocates Memory Row 1 - Waiting...)
   SYN  ------------------------> (Allocates Memory Row 2 - Waiting...)
   SYN  ------------------------> (Allocates Memory Row 3 - Waiting...)
   SYN  ------------------------> (Allocates Memory Row 4 - Waiting...)
                 ...
```

The server obeys the standard laws of networking: for every incoming `[SYN]`, it allocates a small block of its operating system RAM, replies with a `[SYN, ACK]`, and sits waiting. Because the attacker intentionally holds back the final `[ACK]`, those memory allocations remain permanently locked open. 

```text
SYN → SYN, ACK → No final ACK
SYN → SYN, ACK → No final ACK
              ...
              ↓
      Many half-open connections
              ↓
    SYN backlog/resources fill
              ↓
 Legitimate connections fail (504 Gateway Timeout)
```

#### The Corporate Real-World Example (Flipkart Big Billion Days)
Imagine Flipkart is launching its biggest online sale of the year. Within seconds of the sale going live, an attacker hits the web service with an automated flood utility. The tool sends **100,000 initial `[SYN]` packets** per second. 

Flipkart's servers dutifully spin up thousands of half-open connection profiles, consuming its available backend RAM. When an actual customer arrives at the site and clicks "View Products", the server's connection database is 100% full. The genuine customer's browser spins endlessly and eventually returns a connection timeout error.

---

### 💥 2. Understanding DoS vs. DDoS

The core distinction between these two categories rests entirely on the **architecture of the threat source**—specifically, how many unique, physical network interfaces are generating the attack volume.

#### DoS (Denial of Service) — One vs. One
A DoS attack attempts to make a service unavailable using a non-distributed, single network endpoint (one source machine, one network card).
* **🏢 Real-World Example:** A malicious actor sets up an aggressive automated script on their single high-performance cloud server. This single server focuses 100% of its network bandwidth on pumping garbage requests directly into a local company's login page. 
* **Forensic Status:** Because all the traffic flows out of **one single IP address**, a security engineer looking at the firewall logs can write an absolute block rule for that specific source IP, immediately neutralizing the attack.

#### DDoS (Distributed Denial of Service) — Many vs. One
A DDoS attack uses multiple genuine, globally distributed hosts to generate traffic toward the same target.
* **🏢 Real-World Example:** Instead of using one server, a cybercriminal group spends months scanning the internet for unsecured Internet of Things (IoT) devices, such as smart home cameras, enterprise building routers, and internet-connected building thermostats. They infect 200,000 of these innocent devices globally with a malware strain, forming a coordinated puppet army known as a **Botnet**. On the hacker’s command, all 200,000 devices simultaneously stream traffic to Flipkart.
* **Forensic Status:** To the target server's firewall, the incoming traffic appears to be completely legitimate, distributed users coming from thousands of different cities worldwide. The target firewall cannot deploy a simple single-IP block rule without accidentally knocking innocent consumers offline, making a true DDoS exponentially harder to mitigate.

---

### 🎭 3. What is IP Spoofing?

#### Technical Definition
**IP Spoofing** means changing or forging the source IP address inside an internet packet header so that it appears to come from another completely random system.

Every single data packet traveling over the internet functions exactly like an express shipping box. It requires two vital metadata labels stamped into its header:
1. **Destination IP:** The delivery address (where the packet is going).
2. **Source IP:** The return address (where the server should send its replies).

#### The Shadow Sender Trick
When an attacker launches a **Spoofed DoS Attack**, they run a tool that generates thousands of random 4-digit number strings every second and stamps them into the Source IP box:
* Packet 1 Envelope ───> Destination: Target Server ───> Source IP: `1.1.1.1` (Fake)
* Packet 2 Envelope ───> Destination: Target Server ───> Source IP: `99.88.77.66` (Fake)
* Packet 3 Envelope ───> Destination: Target Server ───> Source IP: `210.45.12.93` (Fake)

The standard routers on the internet are simple data traffic controllers; they do not perform identity verification on the Source IP box. They read the destination address and forward the boxes directly to the target. The victim sees thousands of different source IP addresses even though the packets were generated by the exact same attacker machine.

Therefore:
* **SYN Flood + One genuine source** → DoS
* **SYN Flood + Many genuine sources** → DDoS
* **SYN Flood + Spoofed source IPs** → Spoofed SYN Flood (The attack pattern inside this PCAP)

#### 🕵️‍♂️ Unmasking the Lie in Wireshark (IP Spoofing vs. True DDoS)
IP Spoofing allows **one single computer (DoS)** to create a massive wave of traffic that looks exactly like a **global botnet army (DDoS)** because your log columns will display thousands of completely unique IP addresses. 

However, as network forensics investigators, we catch this lie by inspecting **Layer 2 (The Data-Link Layer)**. While an attacker can easily write any fake number they want into the digital IP address field (Layer 3), the physical network interface card sending the packet cannot hide its hardware footprint. 

When you drill down into the packet details in Wireshark and examine the **Ethernet II Source MAC Address**, you will notice a critical anomaly: thousands of different IP addresses are all sharing **one single, identical physical MAC Address**. This absolute fingerprint proves that a single local machine is programmatically forging its network identity labels—confirming a Spoofed DoS event.

---

## 🔬 Part 2: Technical Wireshark Investigation

### 🎯 1. Investigation Objectives
The objective of this investigation is to determine whether the PCAP contains characteristics of a TCP SYN Flood. The investigation focuses on verifying:
- Large numbers of TCP SYN packets targeting a fixed interface.
- The precise destination IP and targeted ports.
- The volume of unique source IPs versus the underlying physical hardware layer.
- Packet arrival rates, timestamps, and TCP handshake completion percentages.

### 2. Environment Setup & Packet Acquisition
* **Step 1 — Download the PCAP:** Download the practice attack trace from [StopDDoS Public Packet Captures: pkt.TCP.synflood.spoofed.pcap](https://githubusercontent.com)
* **Step 2 — Open the PCAP in Wireshark:** Open the downloaded file inside your Wireshark analyzer application.

### 3. Triage Traffic Filter Protocol
We isolate the scanning traffic by focusing purely on the outbound connection requests.
* **Isolating Outbound SYN Probes:** `tcp.flags.syn == 1 && tcp.flags.ack == 0`
* **Checking for Handshake Completions:** `tcp.flags.syn == 0 && tcp.flags.ack == 1`

---

## 🚨 Part 3: Suspicious Indicators Found in PCAP

### Indicator 1 — Thousands of Randomized Spoofed Source IPs
#### Evidence
The packet list pane displays thousands of completely unique, random external IP addresses attacking the system simultaneously. 
#### Forensic Meaning
This confirms **IP Spoofing**. An automated script is bypass-crafting custom packet headers to lie about its network identity, attempting to bypass standard per-IP firewall volume bans.
#### 📸 Wireshark Evidence: Spoofed IP Stream
![](Screenshots/spoofed_ip_stream.png)

---

### Indicator 2 — Multi-Port Variety Shuffling (Non-Sequential Target Focus)
#### Evidence
While the incoming source IP addresses change completely with every single line row, the `Destination IP` and `Destination Port` show the attacker testing a wide variety of completely different application ports in a mixed order. 
#### Forensic Meaning

