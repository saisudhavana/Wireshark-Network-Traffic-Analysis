# DNS Tunneling Case Study: Wireshark Network Forensics

This module documents the step-by-step packet-level investigation of a covert communication channel over DNS. By examining structural variations in name requests and responses, we isolate how malware abuses trusted network paths to bypass enterprise boundaries.

---

## 1. Environment Setup & Packet Acquisition

*   **Step 1:** Download the practice network capture from [Elastic Security's Iodine DNS Tunneling PCAP](https://github.com).
*   **Step 2:** Open the downloaded `dns-tunnel-iodine.pcap` file inside **Wireshark**.
*   **Step 3:** Use the following display filters to separate and triage traffic directions:
    *   **Show All DNS Traffic:** `dns`
    *   **Show Outbound Queries Only:** `dns.flags.response == 0`
    *   **Show Inbound Responses Only:** `dns.flags.response == 1`

---

## 2. Inspecting Outbound DNS Queries
Apply the filter `dns.flags.response == 0` to check outbound network calls for these 5 distinct behavioral indicators:

### Indicator 1: Long DNS Query Names
*   **Observed Evidence:** The captured query strings contain heavily elongated domain paths (e.g., `laegpumiplhhpz12ynd1efljwlkjcgwy.pirate.sea`). The prefix string before the parent domain is unusually long.
*   **Why Suspicious:** Attackers use the query name field to carry stolen data out of the network. A longer subdomain string maximizes the payload space available to transport data chunks.
*   **Wireshark Path:** `Domain Name System` ➔ `Queries` ➔ `Name`

### Indicator 2: Random / Encoded-Looking Subdomains
*   **Observed Evidence:** Captured subdomains consist of unreadable, high-entropy character patterns (e.g., `laegpumiplh...` or `zi03aA-Aaahhh-Drink-mal-ein...`). They do not match normal hostnames like `www`, `mail`, or `api`.
*   **Why Suspicious:** Malware encrypts and scrambles internal corporate files, system hashes, or stolen credentials, then inserts those raw strings straight into the subdomain space.

### Indicator 3: Same Parent Domain with Mutating Subdomains
*   **Observed Evidence:** The capture shows a constant parent zone structure (`pirate.sea`), but the preceding subdomains continuously rotate and shift (e.g., `zi03....pirate.sea`, `zi04....pirate.sea`, `laegpumi....pirate.sea`).
*   **Why Suspicious:** This represents automated file chunking. The malware splits a stolen document or script into sequential blocks and uploads them one request at a time:
    *   *Chunk 1:* `abc.pirate.sea`
    *   *Chunk 2:* `xyz.pirate.sea`
    *   *Chunk 3:* `zi03.pirate.sea`

### Indicator 4: High DNS Query Frequency
*   **Observed Evidence:** Packet logs show a massive volume of requests happening within milliseconds of each other. The timestamps show an automated, rapid rhythm:
    *   `0.000897`
    *   `0.006692`
    *   `0.007103`
    *   `0.007348`
    *   `0.007460`
*   **Why Suspicious:** Human web browsing displays natural, staggered gaps. Bursts of back-to-back requests prove an automated script is continuously running to keep a data transmission channel active.

### Indicator 5: Large Volume of Unique Subdomains
*   **Observed Evidence:** The packet list contains an immense number of completely unique subdomain strings all addressing the single apex target zone `pirate.sea`.
*   **Why Suspicious:** Because every new piece of stolen information requires a fresh query path to exit the network, a massive count of unique subdomains under one zone points directly to active data exfiltration.

---

## 3. Inspecting Inbound DNS Responses
Apply the filter `dns.flags.response == 1` to scrutinize the traffic returning from the external network:

### Indicator 6: Deployment of the `NULL` DNS Record Type
*   **Observed Evidence:** The response packets display `Type: NULL (10)` inside the Answers panel. This takes the place of normal operational entries like `A` (IPv4) or `AAAA` (IPv6).
*   **Why Suspicious:** Healthy web applications never use `NULL` records. `NULL` records have no structural formatting constraints, making them useless for normal internet browsing. Attackers intentionally use them because they act as a blank, unmonitored canvas to move custom raw files past basic network protocol filters.

### Indicator 7: Unexpected Data Payload inside the Response Field
*   **Observed Evidence:** Instead of returning a standard website IP address, the packet's **Answers** section contains a specific field labeled **`Null (data)`** holding custom hexadecimal values.
*   **Why Suspicious:** Normal DNS acts exclusively like an address book to map domain names to web IPs. Seeing raw text or hex data inside a `NULL` record proves that DNS is being weaponized as a **covert data transport channel**. The attacker's Command and Control (C2) server uses this empty response space to slip operational configuration settings and hacker commands right back to the malware inside the network.

---

## 4. Master Investigation Findings Summary

| Indicator Vector | Evidence Observed in PCAP | Technical Significance |
| :--- | :--- | :--- |
| **Long Query Names** | `laegpumiplhhpz12ynd1efljwlkjcgwy.pirate.sea` | Maximizes packet space to smuggle data out. |
| **Random Subdomains** | `zi03...`, `zi04...`, `laegpumi...` | Confirms encrypted data encoding. |
| **Same Parent Domain** | Repeated interactions with `pirate.sea` | Maps directly to rogue attacker infrastructure. |
| **Changing Subdomains** | Continual unique string rotation | Indicates sequential file block chunking. |
| **High Frequency** | Multiple queries generated within milliseconds | Proves automated malware script activity. |
| **Unique Subdomains** | Numerous distinct query names | Confirms systematic data staging. |
| **NULL Record Type** | `NULL (10)` visible in DNS response layers | Evades standard IP formatting constraints. |
| **Response Data Payload** | `Null (data)` field visible in Answers | Smuggles C2 commands back inside. |
| **Packet Hex/ASCII Bytes** | Raw Hex/ASCII data blocks | Contains the tool parameters and text commands. |

### Final Wireshark Conclusion
The reviewed packet capture exhibits definitive, multi-vector indicators of protocol exploitation. The combined presence of long, high-entropy subdomains, rapid query frequencies, structural omissions of valid IP routing information, and unformatted data injections within obsolete `NULL` records confirms an active instance of **Iodine-based DNS Tunneling**.
