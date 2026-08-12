# DNS Tunneling Case Study: Wireshark Network Forensics

This case study documents the packet-level investigation of **DNS tunneling** using Wireshark.

The objective is to identify suspicious DNS behavior and understand how DNS can be abused as a covert communication channel.

---

## 1. Environment Setup & Packet Acquisition

### Step 1 — Download the PCAP

Download the practice PCAP from:

- [Elastic Security's Iodine DNS Tunneling PCAP](https://github.com/elastic/examples/blob/master/Security%20Analytics/dns_tunnel_detection/dns-tunnel-iodine.pcap)

This PCAP contains traffic associated with **iodine DNS tunneling**.

### Step 2 — Open the PCAP in Wireshark

Open the downloaded file:

```text
dns-tunnel-iodine.pcap
````

in **Wireshark**.

---

## 2. DNS Traffic Filtering

Start by displaying all DNS traffic:

```text
dns
```

This allows us to focus only on DNS packets.

We then separate DNS queries and DNS responses.

### DNS Queries

Use:

```text
dns.flags.response == 0
```

This displays DNS requests sent by the client.

### DNS Responses

Use:

```text
dns.flags.response == 1
```

This displays DNS responses sent back to the client.

---

# 3. Inspect DNS Queries

Apply:

```text
dns.flags.response == 0
```

Now inspect the DNS query packets for suspicious indicators.

---

## Indicator 1 — Long DNS Query Names

### Evidence

The PCAP contains long query names such as:

```text
laegpumiplhhpz12ynd1efljwlkjcgwy.pirate.sea
```

The portion before `pirate.sea` is unusually long.

In Wireshark:

```text
Domain Name System
    └── Queries
         └── Name
```

### Why is this suspicious?

Normal DNS queries usually contain short and meaningful hostnames.

In DNS tunneling, the attacker can place encoded or transferred data inside the subdomain.

For example:

```text
<encoded-data>.pirate.sea
```

A longer subdomain provides more space to carry data.

### What we learned

A very long DNS query name is not automatically malicious, but when many long queries appear repeatedly to the same domain, it becomes a strong tunneling indicator.

---

## Indicator 2 — Random / Encoded-Looking Subdomains

### Evidence

The PCAP contains unusual subdomains such as:

```text
laegpumiplhhpz12ynd1efljwlkjcgwy.pirate.sea
```

and:

```text
zi03aA-Aaahhh-Drink-mal-ein-...pirate.sea
```

These do not look like normal hostnames such as:

```text
www.example.com
mail.example.com
api.example.com
```

### Why is this suspicious?

DNS tunneling software can encode information and place the encoded data inside the subdomain.

The result can look:

* random
* encoded
* high-entropy
* difficult to understand
* continuously changing

### What we learned

Random-looking DNS labels are suspicious when they appear repeatedly and are combined with other tunneling indicators.

---

## Indicator 3 — Same Parent Domain with Changing Subdomains

### Evidence

The PCAP repeatedly communicates with:

```text
pirate.sea
```

but the subdomain changes:

```text
zi03....pirate.sea
zi04....pirate.sea
laegpumi....pirate.sea
```

### Why is this suspicious?

A DNS tunnel can divide information into multiple pieces.

Conceptually:

```text
Data chunk 1 → abc.pirate.sea
Data chunk 2 → xyz.pirate.sea
Data chunk 3 → zi03.pirate.sea
```

The parent domain stays the same while the data-containing subdomain changes.

### What we learned

Repeated communication with one domain using many changing subdomains is a strong pattern to investigate for DNS tunneling.

---

## Indicator 4 — High DNS Query Frequency

### Evidence

The packet timestamps show several DNS queries occurring very close together:

```text
0.000897
0.006692
0.007103
0.007348
0.007460
```

The requests are generated within milliseconds.

### Why is this suspicious?

DNS tunneling requires repeated DNS communication to continuously transfer information.

Therefore, tunneling software may generate DNS requests at a high frequency.

### What we learned

High-frequency DNS traffic is not malicious by itself.

However:

```text
High frequency
+
Long queries
+
Changing subdomains
+
Same parent domain
```

creates a much stronger tunneling pattern.

---

## Indicator 5 — Large Number of Unique Subdomains

### Evidence

The same parent domain:

```text
pirate.sea
```

is contacted using many different subdomains.

For example:

```text
zi03....pirate.sea
zi04....pirate.sea
laegpumi....pirate.sea
```

### Why is this suspicious?

DNS tunneling can use a new subdomain for each piece of transferred information.

Therefore, one host generating many unique subdomains under the same domain can indicate that DNS is being used as a data channel.

### What we learned

A high number of unique subdomains is an important DNS tunneling indicator, especially when combined with long and encoded-looking names.

---

# 4. Inspect DNS Responses

Now apply:

```text
dns.flags.response == 1
```

We inspect the responses associated with the suspicious DNS queries.

---

## Indicator 6 — Unusual NULL DNS Record

### Evidence

The PCAP contains:

```text
Type: NULL (10)
```

inside the DNS response.

In the Wireshark packet details, we can see:

```text
Answers
    └── NULL (10)
```

### What is normally expected?

In normal DNS resolution, common response records include:

```text
A      → IPv4 address
AAAA   → IPv6 address
CNAME  → Alias
MX     → Mail server
```

For example:

```text
google.com
      ↓
A record
      ↓
142.250.x.x
```

The DNS response provides information needed to resolve the domain.

### What is different in this PCAP?

Instead of a normal A/AAAA response, we see:

```text
NULL (10)
```

and data associated with the NULL record.

### Why is this suspicious?

A NULL record is unusual in ordinary DNS traffic.

The NULL record can carry arbitrary data rather than simply providing normal DNS resolution information.

In a DNS tunneling scenario, this can be abused to transport information through DNS.

### Simple meaning

Instead of:

```text
Domain → IP address
```

we are seeing:

```text
DNS response → data
```

This is suspicious because DNS is being used to carry information rather than only perform normal name resolution.

---

## Indicator 7 — Data Inside NULL Record

### Evidence

The Wireshark packet details show:

```text
NULL (data)
```

The packet bytes also contain hexadecimal and ASCII data.

For example, the Packet Bytes section contains values such as:

```text
31 33 30 32 33 32 30 32 33 ...
```

### Why is this suspicious?

Normally, we expect a DNS response to contain information such as an IP address.

Here, the response contains data inside a NULL record.

This suggests that the DNS response is being used as a data-carrying channel.

### Simple comparison

Normal DNS:

```text
Client
   ↓
google.com
   ↓
DNS Server
   ↓
A record → IP address
```

Suspicious tunneling behavior:

```text
Client
   ↓
DNS query containing encoded information
   ↓
DNS Server
   ↓
NULL record containing data
```

The combination of the unusual query structure and data-bearing response makes the traffic suspicious for DNS tunneling.

---

# 5. Inspect the Packet Bytes

Wireshark also allows us to inspect the raw packet contents.

In the lower-right **Packet Bytes** section, hexadecimal and ASCII representations of the packet are visible.

Example:

```text
31 33 30 32 33 32 30 ...
```

### Why inspect packet bytes?

Packet bytes allow us to verify what is actually being transmitted at the protocol level.

In this PCAP, the bytes associated with the DNS fields contain data that is not typical of a simple DNS name-resolution exchange.

### Important point

Packet bytes alone do not prove that traffic is malicious.

We use them together with:

* long query names
* unusual subdomains
* repeated queries
* high query frequency
* many unique subdomains
* unusual NULL records
* data inside DNS responses

to build the detection conclusion.

---

# 6. Wireshark Investigation Summary

| Indicator                 | Evidence in PCAP                              | Why Suspicious                                          |
| ------------------------- | --------------------------------------------- | ------------------------------------------------------- |
| Long DNS query names      | `laegpumiplhhpz12ynd1efljwlkjcgwy.pirate.sea` | Provides more space for encoded/transported data        |
| Random-looking subdomains | `zi03...`, `zi04...`, `laegpumi...`           | May contain encoded information                         |
| Same parent domain        | Repeated `pirate.sea`                         | Indicates repeated communication with the same DNS zone |
| Changing subdomains       | Different subdomains under `pirate.sea`       | Can represent different data chunks                     |
| High query frequency      | Multiple queries within milliseconds          | Consistent with automated tunneling activity            |
| Unique subdomains         | Many different subdomain values               | Common tunneling behavior                               |
| NULL record               | `NULL (10)`                                   | Unusual DNS response type that can carry arbitrary data |
| Response data             | `NULL (data)`                                 | Indicates data is being carried in the DNS response     |
| Packet bytes              | Hex/ASCII data visible                        | Allows verification of the underlying DNS payload       |

---

# 7. Final Wireshark Conclusion

The PCAP shows several indicators consistent with **DNS tunneling**.

The most important evidence is the combination of:

```text
Long DNS query names
        +
Random/encoded-looking subdomains
        +
Same parent domain
        +
Changing subdomains
        +
High query frequency
        +
Many unique subdomains
        +
NULL DNS records
        +
Data inside DNS responses
```

Individually, these indicators may have legitimate explanations.

However, when they occur together in the same communication flow, they strongly indicate that DNS is being used as a covert data communication channel.

The capture is associated with **iodine-based DNS tunneling**.

---

# 8. MITRE ATT&CK Mapping

### Tactic

**Command and Control**

### Technique

**T1071.004 — Application Layer Protocol: DNS**

Attackers can abuse DNS to communicate with external infrastructure while blending their traffic with normal DNS activity.

### DNS Tunneling

DNS tunneling is commonly associated with:

```text
T1071.004 — DNS
```

The technique covers abuse of DNS as an application-layer communication protocol.

---

# 9. Key Interview Takeaway

If asked:

**"How would you investigate DNS tunneling in Wireshark?"**

A good answer is:

> "I would first filter DNS traffic using `dns`. Then I would separate queries and responses using `dns.flags.response == 0` and `dns.flags.response == 1`. I would look for long and random-looking subdomains, repeated queries to the same parent domain, many unique subdomains, high query frequency, unusual DNS record types, and data carried inside DNS responses. In this PCAP, the combination of changing long subdomains and NULL records containing data is consistent with iodine-based DNS tunneling."

---

# 10. Evidence Collected

The investigation provides the following evidence:

* DNS query filter
* DNS response filter
* Long query names
* Random/encoded-looking subdomains
* Repeated `pirate.sea` parent domain
* Changing subdomains
* High-frequency DNS queries
* Multiple unique subdomains
* NULL DNS records
* Data inside NULL responses
* Packet-level hexadecimal/ASCII evidence

These screenshots can be added to the GitHub repository under:

```text
screenshots/
└── dns-tunneling/
    ├── 01-dns-filter.png
    ├── 02-long-query.png
    ├── 03-changing-subdomains.png
    ├── 04-high-frequency.png
    ├── 05-null-record.png
    └── 06-response-data.png
```

---

## Conclusion

This investigation demonstrates how Wireshark can be used to identify DNS tunneling by analyzing DNS packet structure, query behavior, response records, and packet contents.

The key lesson is:

> **DNS tunneling is not identified by one indicator. It is identified by the combination of unusual DNS behavior and data patterns.**

```
```
