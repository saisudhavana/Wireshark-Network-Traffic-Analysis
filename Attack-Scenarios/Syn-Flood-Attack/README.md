# 🚪 TCP SYN Flood Attack Investigation

This project explains how a **TCP SYN Flood attack** works and demonstrates how to investigate the attack using **Wireshark and a PCAP file**.

The investigation focuses on identifying suspicious SYN traffic, understanding the TCP handshake, analyzing source and destination IPs, checking incomplete connections, and determining whether IP spoofing or a distributed attack may be involved.

---

# 📖 1. What is a TCP SYN Flood?

## Simple Definition

A **TCP SYN Flood** is a **Denial-of-Service (DoS) attack** that abuses the TCP connection establishment process.

The attacker sends a large number of TCP `SYN` packets to a server but does not complete the TCP 3-way handshake.

When the server receives a SYN, it prepares and maintains information about the connection in a **half-open state** while waiting for the final `ACK`.

If too many half-open connections are created, the server's **SYN backlog or other connection resources can become exhausted**.

As a result, legitimate users may experience slow connections, connection failures, or timeouts.

---

# 🔄 2. How Does the TCP 3-Way Handshake Work?

A normal TCP connection uses three steps:

```text
Client                         Server

   SYN  ------------------------>

        <------------------  SYN/ACK

   ACK  ------------------------>

         Connection Established
