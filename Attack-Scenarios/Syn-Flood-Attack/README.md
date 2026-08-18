# 🚪 TCP SYN Flood Attack Investigation Case Study

This project explains how a **TCP SYN Flood attack** works and how to identify it using Wireshark. 

---

## 📖 Part 1: Core Cyber Security Concepts (Theory)

### 💡 1. What is a TCP SYN Flood?

#### Simple Definition
A **TCP SYN Flood** is an attack that crashes a computer server by breaking the standard rules of how computers say hello to each other. 

#### How a Normal TCP Connection Works
When your computer wants to talk to a server (like your web browser opening Flipkart), they must perform a **3-Way Handshake**. This is a simple 3-step greeting:
*   **Step 1 (SYN):** Your browser knocks on the server's door and says, *"Hello! I want to open your website."*
*   **Step 2 (SYN/ACK):** The server answers back, *"Hello! I see you. I have saved a small spot in my memory for you. Let me know when you are ready."* (This is a **half-open connection**).
*   **Step 3 (ACK):** Your browser answers, *"Got it! Let's start loading the page."* 

Once Step 3 happens, the connection is fully open, and you can buy products.

#### What Happens During a SYN Flood?
During a SYN Flood, an attacker uses a hacking program to repeat Step 1 millions of times. The script fires a massive wave of `SYN` packets at the server but **never sends the final `ACK` packet (Step 3)**.

The server obeys the rules of the internet. For every single request, it opens a half-open connection slot and wastes its memory waiting for an answer. 

Because the attacker never finishes the handshake, the server's memory slots (called the **SYN backlog**) get completely filled with fake, incomplete connections. 

When a genuine customer tries to visit the website, the server is out of memory slots. The website spins endlessly and crashes.

#### The Real-World Example (Flipkart Big Billion Days)
Imagine Flipkart is launching a massive online sale. 
Under normal conditions, customers knock on the door, get a memory slot, and complete their purchases safely. 

Now, an attacker enters the picture. The attacker uses a tool to send **100,000 fake reservation requests** per second. Flipkart's servers dutifully reserve slots in their RAM for all of them. 

When an actual customer arrives and clicks "Buy Now", the connection database is 100% full of fake requests. The real customer gets blocked, receives a connection error, and cannot buy anything.

---

### 💥 2. Understanding DoS vs. DDoS

The difference between these two types of attacks comes down to a simple question: **How many physical computers are attacking the server?**

#### DoS (Denial of Service) — One vs. One
A DoS attack happens when **one single attacker computer** floods a target server to take it down.
*   **Real-World Example:** One person sets up an aggressive automated script on their single laptop. They focus 100% of their internet speed on flooding a local business website.
*   **How to stop it:** This is very easy to fix. A network engineer looks at the security logs, sees that a single IP address is causing the entire flood, and blocks that one IP with a simple firewall rule. The attack stops instantly.

#### DDoS (Distributed Denial of Service) — Many vs. One
A DDoS attack happens when **thousands of different computers** across the world attack a target at the exact same time.
*   **Real-World Example:** The hacker does not use their own laptop. Instead, they infect thousands of everyday internet devices (like smart TVs, home routers, and security cameras) with malware. This secret zombie army is called a **Botnet**. On the hacker's command, all these thousands of devices start flooding Flipkart simultaneously.
*   **How to stop it:** This is incredibly difficult to defend against. The traffic comes from thousands of unique, real IP addresses globally. You cannot just block one IP. If you block everything blindly, you will accidentally block real, innocent customers.

---

### 🎭 3. What is IP Spoofing?

#### Simple Definition
**IP Spoofing** means **lying about your identity**. It is the act of falsifying the sender's IP address inside a network packet.

Every piece of data sent over the internet functions like a letter inside a mailing envelope. It needs two vital labels:
1.  **Destination IP:** Where the packet is going (Flipkart's server).
2.  **Source IP:** The return address (where the server should send its replies).

When a hacker launches a **Spoofed SYN Flood**, they use a hacking tool that automatically generates random, fake numbers every second and stamps them into the Source IP box. 

The internet routers do not check if the return address is real. They simply read the destination address and deliver the packets to the server. The victim server sees thousands of different source IP addresses in its logs, even though **only one attacker machine** is making them.

#### The Formulas:
*   **SYN Flood + One real source IP** = Standard DoS
*   **SYN Flood + Many real source devices** = True DDoS
*   **SYN Flood + Fake source IPs** = Spoofed SYN Flood (The attack inside this PCAP)

#### 🕵️‍♂️ The Forensic Trick: Finding the Truth in Wireshark
Because IP Spoofing creates thousands of fake IP addresses in your logs, a basic DoS attack can trick you into thinking it is a massive global DDoS botnet. 

However, network investigators can easily catch this lie by checking **Layer 2 (The Hardware MAC Address)**. 

An attacker can type any fake number they want into the digital IP address field (Layer 3), but the physical network card inside their computer cannot hide its unique hardware signature. 

When you look into the packet details in Wireshark and examine the **Source MAC Address**, you will notice a major red flag: thousands of completely different IP addresses are all sharing **one single, identical hardware MAC address**. This absolute fingerprint proves that a single physical machine is lying about its identity—confirming it is a Spoofed DoS attack.

---

## 🔬 Part 2: Technical Wireshark Investigation

### 🎯 1. Investigation Objectives
The goal of this practical lab is to look inside a real packet capture file (`.pcap`) and verify if a TCP SYN Flood is happening. We will look for:
*   A massive volume of `SYN` packets targeting one server.
*   The exact destination port being flooded.
*   Whether the source IP addresses are fake or real by looking at the hardware layer.
*   Whether the server is receiving replies or dealing with total silence.

### 2. Environment Setup & Packet Acquisition
*   **Step 1 — Download the PCAP:** Grab the real attack trace file from [StopDDoS Public Packet Captures](https://githubusercontent.com)
*   **Step 2 — Open the PCAP in Wireshark:** Double-click the file to open it inside your Wireshark traffic analyzer.

### 3. Traffic Filter Cheat Sheet
Type these simple commands into the Wireshark filter bar to isolate the attack data:
*   **Show only incoming scan/flood packets:** `tcp.flags.syn == 1 && tcp.flags.ack == 0`
*   **Check if any handshakes finished:** `tcp.flags.syn == 0 && tcp.flags.ack == 1`

---

## 🚨 Part 4: Suspicious Indicators Found in PCAP

### Indicator 1 — Thousands of Fake Source IPs
*   **Evidence:** The packet list pane displays thousands of completely unique, random external IP addresses hitting the system at the same time.
*   **Meaning:** This is **IP Spoofing**. An automated script is creating fake packet labels to hide its real location and trick basic firewall filters.
*   **📸 Screenshot Link:** `![](Screenshots/spoofed_ip_stream.png)`

---

### Indicator 2 — Heavy Attack Volume on a Single Port
*   **Evidence:** While the source IP addresses change completely on every single line, the `Destination IP` and `Destination Port` stay 100% frozen on the victim server.
*   **Meaning:** Real web browsers talk to multiple ports or load background links. Having thousands of random IPs target one single application socket confirms a deliberate resource exhaustion attack.
*   **📸 Screenshot Link:** `![](Screenshots/single_port_focus.png)`

---

### Indicator 3 — Impossibly Fast Robotic Pacing
*   **Evidence:** Looking at the `Time` column, packets are hitting the network card at an extreme speed of several hundred requests per microsecond.
*   **Meaning:** Humans have natural typing delays and variable loading pauses. The complete absence of timing variation proves an automated hacking loop is flooding the system.
*   **📸 Screenshot Link:** `![](Screenshots/packet_burst_timestamps.png)`

---

### Indicator 4 — One-Way Traffic (Total Silence from the Target)
*   **Evidence:** When looking at the entire file, there are **only incoming `[SYN]` packets**. There are zero responses or handshakes coming back from the target machine.
*   **Meaning:** This proves the target server is either completely dead from the flood, or its firewall is successfully **dropping/ignoring** every packet silently. The firewall created a total wall of silence to block the hacker from learning anything.
*   **📸 Screenshot Link:** `![](Screenshots/rst_responses.png)`

---

### Indicator 5 — Hardware Layer Duplication (The Smoking Gun Proof)
*   **Evidence:** When you click through multiple packets with different IP addresses, the **Ethernet II -> Source MAC Address** field stays completely identical.
    *   **Source Hardware MAC Card:** `44:f4:77:0f:ea:49` (Juniper Networks interface)
*   **Meaning:** This is the ultimate proof of a **Spoofed DoS Attack**. The hacking tool successfully changed the digital IP address on every packet, but it could not hide the physical network card. One MAC address means one single computer is generating the entire flood.
*   **📸 Screenshot Link:** `![](Screenshots/hardware_mac_unmask.png)`

---

## 📊 Part 5: Wireshark Investigation Summary

| Indicator | Evidence in PCAP | Simple Forensic Meaning |
| :--- | :--- | :--- |
