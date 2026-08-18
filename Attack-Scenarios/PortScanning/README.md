#  Port Scanning Fundamentals: Cyber Reconnaissance Explained

##  What is Port Scanning?
Every computer server on the internet has a single IP address (like a street address), but it needs to do many different jobs at once. To stay organized, a server has **65,535 virtual communication doors** called **Ports**.
*   **Port 443:** The secure door used for web browsing (HTTPS).
*   **Port 22:** The locked backend door used by system engineers to manage the server (SSH).

**Port Scanning** is an automated method where an attacker runs a software tool to knock on thousands of these virtual doors, one after another, in just a few seconds to see which ones are open, closed, or blocked.

### 🏢 The Real-World Example
Think of a server as a **massive shopping mall** at 3:00 AM. A port scan is the equivalent of a burglar walking down the hallway and twisting every single shop doorknob to find an unlocked store.

---

##  Why and How Attackers Abuse It?

Attackers cannot simply "hack a computer" blindly. They must target a **specific software application** running behind an open door on that computer. 

### The Hacker Kill Chain (Step-by-Step)
1. **Discovery:** The attacker runs a scan and finds out that **Port 443** is wide open.
2. **Peeking Inside (Banner Grabbing):** They send a basic message to Port 443. The software inside introduces itself: *"Hello! I am Apache Web Server Version 2.4.41."*
3. **Hunting for Glitches (CVE Lookup):** The hacker goes to a public database of known software bugs. They type in *"Apache 2.4.41"* and find a critical vulnerability called **CVE-2021-41773** (a glitch that lets outsiders read secret files).
4. **The Abuse (Exploitation):** Because Port 443 is open to the public, the hacker sends a custom, corrupted web request designed to trigger that exact bug. The software crashes, giving the hacker total control of the system.

---

## 🚨 The Suspicious Indicators (What We Hunt For)

Normal internet traffic looks like a friendly, slow conversation. Port scanning looks completely unnatural. When analyzing network logs, we hunt for these **4 major red flags**:

### 1. The One-to-Many Profile (High Volume)
*   **Definition:** A single computer tries to talk to thousands of different ports on a target server in a fraction of a second.
*   **Example:** A regular user's browser only opens Port 80 or 443 to view a website. If a single IP address suddenly tries to talk to Port 21, then 22, then 23, then 25, then 80 sequentially, it is an automated scanner.

### 2. Broken Conversations (The Half-Open Trick)
*   **Definition:** The attacker asks to connect, but the exact millisecond the server says *"Yes, I am open!"*, the attacker slams the door shut using a **Reset (`RST`)** packet instead of finishing the handshake.
*   **Example:** It is like walking up to a house, ringing the doorbell, and running away the moment someone answers. Attackers do this on purpose because half-open connections often don't get recorded in normal application logs, keeping them "invisible."

### 3. Robotic Timing (The Constant Pacing)
*   **Definition:** Packets hit the server at an absolutely flawless, unvarying mathematical speed.
*   **Example:** Humans have chaotic typing speeds. If network logs show a connection request arriving exactly every **100 milliseconds** (0.10s, 0.20s, 0.30s) without a single millisecond of variation, it is a script or automated machine operating the attack.

### 4. Administrative Gate Hunting
*   **Definition:** The traffic shows an immediate, heavy focus on ports used purely for backend network management.
*   **Example:** A public customer on Flipkart has no business touching **Port 22 (SSH)** or **Port 3389 (Remote Desktop)**. When an unknown external IP address is caught probing these specific ports, it indicates a threat actor hunting for a backdoor to take over the network infrastructure.

---
# 🔬 Part 2: Technical Wireshark Investigation

## 1. Environment Setup & Packet Acquisition

### Step 1 — Download the PCAP
Download the practice reconnaissance capture file from:
- [Practical Packet Analysis Portscan Trace]([https://githubusercontent.com](https://github.com/markofu/pcaps/blob/master/PracticalPacketAnalysis/ppa-capture-files/portscan.pcap)+)

### Step 2 — Open the PCAP in Wireshark
Open the downloaded file `portscan.pcap` inside **Wireshark**.

---

## 2. Port Scan Traffic Filtering

We isolate the scanning traffic by focusing purely on the outbound connection requests.

### Outbound Connection Probes (SYN Queries)
Use this filter to see all requests sent by the attacker:
```text
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

---

## 3. Inspecting Port Scan Indicators

### Indicator 1 — Massive Volume of Outbound SYN Requests
#### Evidence
The packet list pane shows a single source IP address (`10.100.25.14`) rapidly sending thousands of outbound `[SYN]` packets to a single destination IP (`10.100.18.12`).

#### Wireshark Path
```text
Transmission Control Protocol └── Flags └── .... ..1. .... = Syn: Set
```

#### 📸 Wireshark Evidence: Outbound Scan Volume
![](/Attack-Scenarios/PortScanning/Screenshots/Port_Scanning.png)

---

### Indicator 2 — Multi-Port Variety Shuffling (Non-Sequential)
#### Evidence
The `Destination Port` column shows the attacker testing a wide variety of completely different application ports (like **139, 135, 445, 80, 22, 23, 21**) in a mixed order. 

#### Why is this suspicious?
A regular user opens a connection to just one or two ports (like 443 for browsing). A single computer hitting a massive variety of different backend services all at once proves it is a scanner searching for any possible weak spot.

#### 📸 Wireshark Evidence: Diverse Port Probing
![](/Attack-Scenarios/PortScanning/Screenshots/Ports.png)

---

### Indicator 3 — Uniform Robotic Timestamps
#### Evidence
The packet timestamps and the `Time Delta` column show requests arriving at a perfectly fixed mathematical speed of exactly **0.10 seconds (100 milliseconds)** apart.

#### 📸 Wireshark Evidence: Delta Time Analysis
![](/Attack-Scenarios/PortScanning/Screenshots/Ports.png)

---

## 4. Inspecting the Port Feedback Loop (The Filtered Verdict)

### Indicator 4 — One-Way Traffic Stream (Zero Server Responses)
#### Evidence
When looking at the entire packet capture, there are **only outbound `[SYN]` packets**. There are zero incoming `[SYN, ACK]` packets and zero incoming `[RST]` packets from the target machine.

#### Why is this important to mention?
This proves the attack failed! Because the server or its firewall completely **dropped and ignored** every single packet, the hacker received a total wall of silence. The attacker learned zero software version numbers and zero open ports. The network defense successfully stopped the reconnaissance.

#### 📸 Wireshark Evidence: Firewall Dropping Probes
![](/Attack-Scenarios/PortScanning/Screenshots/portscanning_SynPacket.png)

---

## 5. Wireshark Investigation Summary

| Indicator | Evidence in PCAP | Why Suspicious / Forensic Meaning |
| :--- | :--- | :--- |
| **Monolithic SYN Streams** | `tcp.flags.syn==1 && tcp.flags.ack==0` | High-volume outbound requests missing completed handshakes. |
| **Multi-Port Shuffling** | Variety of ports tested (135, 445, 80, 22) | Confirms an active probe scanning across different network services. |
| **Fixed Time Cadence** | `0.10s` intervals between packets | Proves programmatic script automation over normal human traffic. |
| **One-Way Traffic (No Replies)** | 100% `SYN` packets, 0% Replies | Hard forensic evidence that a firewall is actively filtering and dropping the scan. |

