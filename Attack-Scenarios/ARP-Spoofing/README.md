#  ARP Spoofing & Adversary-in-the-Middle Investigation

This case study documents the network-layer mechanics, cache architecture flaws, and packet-level forensic investigation of an active **ARP Spoofing / Man-in-the-Middle (MITM)** event using Wireshark.

---

## 📖 Part 1: Core Cyber Security Concepts (Theory)

### 1. Definition

**ARP (Address Resolution Protocol) spoofing** is a Layer 2 network attack where an attacker sends **forged ARP messages** to associate their own MAC address with the IP address of another device, typically the **default gateway**.

The victim then sends traffic intended for the legitimate gateway to the attacker's MAC address.

#### Normal communication
```text
Victim
192.168.1.10
     |
     | 192.168.1.1 → Router MAC
     ↓
Router
192.168.1.1
```

#### After ARP spoofing
```text
Victim
192.168.1.10
     |
     | 192.168.1.1 → Attacker MAC
     ↓
Attacker
192.168.1.50
     |
     ↓
Router
192.168.1.1
```
The attacker has effectively positioned themselves between the victim and gateway.

---

### 2. How the Attacker Abuses ARP

ARP does not inherently authenticate ARP replies. An attacker on the same local network can therefore send forged ARP responses.

#### Example Scenario
Assume the following host configurations on the local subnet:
```text
Victim Workstation:  IP = 192.168.1.10   | MAC = VICTIM-MAC
Default Router:      IP = 192.168.1.1    | MAC = ROUTER-MAC
Attacker Machine:    IP = 192.168.1.50   | MAC = ATTACKER-MAC
```

Under normal operational baselines:
```text
192.168.1.1 → ROUTER-MAC
```

The attacker logs onto the segment and transmits a forged ARP message, forcing the victim device to update its records:
```text
192.168.1.1 → ATTACKER-MAC
```
The victim now incorrectly believes that **`192.168.1.1` is located at the attacker's MAC address**.

The attacker can then potentially **intercept, modify, redirect, or disrupt** traffic. If the attacker routes the frames cleanly back to the real default gateway, this establishes a functional **Man-in-the-Middle (MITM) attack**:

```text
Victim ──> Attacker ──> Router
Router ──> Attacker ──> Victim
```
> 🛡️ **Note:** Encryption protocols such as HTTPS/TLS will still protect application-layer payloads; ARP spoofing by itself does **not** decrypt HTTPS.

---

### 3. Why Attackers Use ARP Spoofing
ARP spoofing can be abused to achieve several malicious target operations:
* Perform **Man-in-the-Middle (MITM) attacks**
* Intercept internal broadcast network traffic
* Modify unencrypted plaintext traffic in mid-transit
* Intentionally redirect routing tables
---

## 💥 Part 2: ARP Spoofing vs. ARP Poisoning

These terms are often used interchangeably, but there is a distinct difference between the action and the result:

| ARP Spoofing | ARP Poisoning |
| :--- | :--- |
| Forging ARP information | Corrupting/altering a device's ARP cache with false information |
| Focuses on the **attacker's technique** | Focuses on the **effect on the victim** |
| Example: attacker claims `192.168.1.1 → ATTACKER-MAC` | Victim's cache becomes `192.168.1.1 → ATTACKER-MAC` |
| Used to deceive hosts | Result is incorrect IP-to-MAC mapping |

### Simple way to remember
> **Spoofing** = the attacker sends the fake information.
> **Poisoning** = the victim's ARP cache accepts the fake information.

*In practical environment reporting, "ARP spoofing" and "ARP poisoning" are frequently used together to describe the exact same cyber security incident.*

---

## 🛡️ Part 3: MITRE ATT&CK Mapping

ARP spoofing is formally mapped under the enterprise framework matrix:

**MITRE ATT&CK Enterprise — T1557: Adversary-in-the-Middle**

This technique covers attacks where an adversary positions themselves between two communicating systems to intercept or manipulate local network traffic.

### Relevant sub-technique
**T1557.002 — ARP Cache Poisoning**
This specifically covers the strategic manipulation of ARP cache allocation parameters to redirect local network traffic through an adversary-controlled host system interface.

### Attack chain
```text
ARP Spoofing
     ↓
ARP Cache Poisoning
     ↓
Traffic redirected
     ↓
Attacker positioned between hosts
     ↓
Adversary-in-the-Middle
     ↓
Possible interception / modification
```

---

## 🔍 Part 4: SOC Investigation Indicators

When analyzing a suspicious local area network capture inside Wireshark, investigators hunt for these key indicators:
* The same logical IP address associated with multiple distinct hardware MAC addresses simultaneously.
* The default gateway IP suddenly mapped to an unexpected, unverified host MAC address.
* Waves of continuous, unsolicited ARP replies entering the line without any preceding queries.
* One single physical MAC address aggressively claiming ownership of multiple unique local IP address indices.

> 📊 **Key evidence:** If `192.168.1.1` (the gateway) legitimately maps to `MAC-A`, but the PCAP shows it being repeatedly advertised as `MAC-B`, where `MAC-B` belongs to a regular employee host machine, that is a definitive ARP-spoofing indicator.

---
# 🔬 Part 2: Technical Wireshark Investigation

## 1. Environment Setup & Packet Acquisition
*   **Step 1 — Download the PCAP:** Download the practice file from the Wireshark website: [Direct Download Link: arpspoof.pcap](https://github.com/researcher111/ARP-pcap-files/blob/master/arpspoof.pcap)
*   **Step 2 — Open the PCAP in Wireshark:** Open the downloaded file `arp-storm.pcap` inside **Wireshark**.

---

## 2. Running the Wireshark Filters

To find the attack packets, type these simple commands one by one into the top bar of Wireshark:

### 2.1 The Baseline ARP Filter
```text
arp
```
*   **What it does:** This shows *only* the Address Resolution Protocol traffic and hides everything else. It lets us see the massive storm of routing packets filling the network.
  #### 📸 Wireshark Evidence:![]( /Attack-Scenarios/ARP-Spoofing/Screenshots/arp-1.png)
  

### 2.2 Separating Requests and Replies
*   **`arp.opcode == 1` (ARP Request):** This shows packets where a computer is asking a question to the network: *"Who has this IP address? Please tell me your MAC address."*
       #### 📸 Wireshark Evidence:![]( /Attack-Scenarios/ARP-Spoofing/Screenshots/arp-2.png)
    
*   **`arp.opcode == 2` (ARP Reply):** This shows packets where a computer is giving an answer: *"I have that IP address, and this is my physical MAC address."*

     #### 📸 Wireshark Evidence:![]( /Attack-Scenarios/ARP-Spoofing/Screenshots/arp-3.png)
    
### 2.3 The Error Detection Filter
```text
arp.duplicate-address-detected
```
*   **What it does:** This shows packets where Wireshark caught an error or an attack. It instantly highlights lines in **Yellow and Black** because something is wrong with the network layout.
#### 📸 Wireshark Evidence:![]( /Attack-Scenarios/ARP-Spoofing/Screenshots/arp-34.png)

---

## 3. Suspicious Indicators & Packet Analysis

By looking closely at the file, your 2-hour investigation reveals the exact steps of how the attacker tricked the network.

### 3.1 Analyzing Packet 22 (The Attacker's First Move)
When we open **Packet 22**, we can see a normal computer introduction on the network:

#### 📸 Wireshark Evidence:![]( /Attack-Scenarios/ARP-Spoofing/Screenshots/arp-5.png)

#### What you see in the columns:
*   **Source Column:** This shows **`PCSSystemtec_2d:f8:5a`**. This is the real, physical network card address (MAC address) of the machine sending the packet.
*   **Info Column:** This reads **`Who has 192.168.1.1? Tell 192.168.1.105`**
    *   *What this means:* The computer sitting at IP address `192.168.1.105` is shouting a question to everyone: *"Who owns the internet router IP address `192.168.1.1`? If you are the router, please reply directly back to me at my IP address `192.168.1.105`."*

#### What you see in the Middle Pane (ARP Header details):
*   **Sender IP Address:** `192.168.1.105` (The computer asking the question).
*   **Sender MAC Address:** `08:00:27:2d:f8:5a` (The real physical face of this computer).
*   **Target IP Address:** `192.168.1.1` (The router it is looking for).
*   **Target MAC Address:** `00:00:00:00:00:00`
    *   *Why it is all zeros:* Because this is a request question, the computer **does not know** the router's physical card address yet. It leaves this box blank with zeros so the network switches will pass the question around until the real router responds.
 
 #### 📸 Wireshark Evidence:![]( /Attack-Scenarios/ARP-Spoofing/Screenshots/arp-6.png)

---

### 3.2 Analyzing Packet 33 (The Victim's Move)
Next, let's look at **Packet 33** to see another computer on the same network:

#### 📸 Wireshark Evidence:![]( /Attack-Scenarios/ARP-Spoofing/Screenshots/arp-7.png)

#### What you see in the columns:
*   **Source Column:** This shows **`PCSSystemtec_b8:b7:58`**.
*   **Info Column:** This reads **`Who has 192.168.1.1? Tell 192.168.1.104`**
    *   *What this means:* A completely separate, innocent user computer sitting at IP address `192.168.1.104` is asking the exact same question to find the router.

#### What you see in the Middle Pane (ARP Header details):
*   **Sender IP Address:** `192.168.1.104` (The innocent computer asking the question).
*   **Sender MAC Address:** `08:00:27:b8:b7:58` (Its real physical network card signature).
*   **Target IP Address:** `192.168.1.1` (The router it is looking for).
*   **Target MAC Address:** `00:00:00:00:00:00` (Left blank as zeros because it is waiting for an answer).


---

### 3.3 The Baseline Identity Table

By looking at these two early packets, we can build a clean table to remember who owns which IP address and MAC address before the attack starts:

| Packet No. | Type of Packet | Sender IP | Sender MAC (Physical Face) | Target IP | Target MAC |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **22** | Request Query | `192.168.1.105` | `08:00:27:2d:f8:5a` | `192.168.1.1` | `00:00:00:00:00:00` |
| **33** | Request Query | `192.168.1.104` | `08:00:27:b8:b7:58` | `192.168.1.1` | `00:00:00:00:00:00` |

---

### 3.4 Dissecting Packet 3564 (The Active Hacking Identity Theft)

In **Packet 3564**, the attack officially happens. This packet is highly dangerous and catches the hacker red-handed:

#### 📸 Wireshark Evidence:![]( /Attack-Scenarios/ARP-Spoofing/Screenshots/arp-8.png)

#### Why it is malicious and how the forgery happened:
The packet text inside the Info column claims to be a normal answer from the internet router, saying: **`192.168.1.1 is at 08:00:27:2d:f8:5a`**.

But when we check the **Source MAC Address column**, the physical network card code says **`PCSSystemtec_2d:f8:5a`**. 

By looking back at our baseline table, we instantly catch the lie:
*   The real router (`192.168.1.1`) is actually at MAC address ending in **`7c`** (verified in Packet 35).
*   The MAC address ending in **`5a`** belongs to the workstation at IP **`192.168.1.105`** (verified in Packet 22).

#### Unmasking the Disguise:
The attacker at IP `192.168.1.105` is using a hacking tool to lie. The script types the router's IP address (**`192.168.1.1`**) into the `Sender IP` box to pretend to be the router. But because it is sending the packet from its own hardware card, its real physical MAC address (**`2d:f8:5a`**) stays printed on the packet.

The attacker sends this fake answer directly to **Target IP `192.168.1.104`** (The Victim). The attacker is telling the victim machine: *"Hey `.104`, I am the internet router. Send all your private web traffic directly to my MAC address ending in `5a`."* 

Because the attacker typed `192.168.1.1` into the sender box, **the attacker's real IP address (`192.168.1.105`) completely disappears from the text columns.** The only IP addresses you can see left inside this packet row are the fake router address (`.1`) and the victim getting fooled (**`.104`**).

#### Solving the Riddle: Why does it say "Duplicate IP Detected" instead of "Duplicate MAC"?
You noticed that Wireshark pops up a bright yellow warning that says:
`[Duplicate IP address detected for 192.168.1.1 (08:00:27:2d:f8:5a)]`

It is easy to think this should be called a "Duplicate MAC" warning, but Wireshark uses the word **Duplicate IP** because of a basic rule of networks: an IP address can only belong to one physical machine at a time.
*   Earlier in the file, Wireshark recorded that IP `192.168.1.1` belonged to the real router MAC card ending in **`7c`**.
*   Now, in Packet 3564, it sees a new packet claiming that the exact same IP address `192.168.1.1` is at the attacker's MAC card ending in **`5a`**.

Wireshark is warning you that **two entirely different physical computers are claiming they own the exact same IP address (`192.168.1.1`) at the same time**. Because the IP address identity has been cloned across two separate devices on the same network, Wireshark flags it as an infrastructure-level **Duplicate IP Conflict**.

#### 📸 Wireshark Evidence:
![]( /Attack-Scenarios/ARP-Spoofing/Screenshots/arp-34.png)

---

##  Investigation Conclusion

The packet-level forensic analysis of `arp-storm.pcap` provides absolute, definitive proof of a successful **Adversary-in-the-Middle (MITM) attack** driven by local network cache poisoning. 

### Final Forensic Verdict
1. **The Attacker:** Host machine **`192.168.1.105`** (`08:00:27:2d:f8:5a`) successfully hijacked local routing channels.
2. **The Victim:** Host machine **`192.168.1.104`** (`08:00:27:b8:b7:58`) was completely fooled into routing its data to the wrong node.
3. **The Compromise:** By sending thousands of unauthorized, fake ARP replies, the attacker effectively stole the identity of the default gateway router (**`192.168.1.1`**). 

The investigation successfully validated all target parameters by unmasking the core structural mismatch between Layer-3 digital IP addresses and Layer-2 physical hardware MAC signatures. The perimeter controls successfully logged the active collision, triggering the enterprise-level `Duplicate IP address detected` warning flag inside Wireshark's expert tracking engine.

---


