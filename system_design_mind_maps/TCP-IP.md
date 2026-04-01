# TCP/IP Protocol Mind Map

## 1. Introduction
- **TCP/IP**: A suite of communication protocols used to connect devices on the internet.
- **Purpose**: Ensures data travels from one computer to another across networks.

---

## 2. TCP/IP Model Layers
- **1. Application Layer**
  - Provides services like email (SMTP), web browsing (HTTP/HTTPS), file transfer (FTP).
  - **Protocols**: HTTP, FTP, SMTP, DNS, Telnet.
  
- **2. Transport Layer**
  - Ensures reliable data transfer between devices.
  - **TCP**: Reliable, connection-based.
  - **UDP**: Faster, but no guarantee of delivery (connectionless).

- **3. Internet Layer**
  - Manages addressing and routing of packets across networks.
  - **Protocols**: IPv4, IPv6, ICMP, ARP.

- **4. Network Interface Layer**
  - Responsible for physical transmission of data on a network.
  - **Includes**: Ethernet, Wi-Fi, and MAC addresses.

---

## 3. TCP vs. UDP
- **TCP (Transmission Control Protocol)**:
  - Reliable, ordered, and error-checked delivery.
  - Slower due to acknowledgment of each packet.
  - Example: Web browsing (HTTP), email (SMTP).

- **UDP (User Datagram Protocol)**:
  - Faster but no guarantee of delivery.
  - Used for time-sensitive applications like streaming.
  - Example: Video streaming, VoIP.

---

## 4. Addressing and Routing
- **IP Addresses**: Unique identifiers for devices on the network.
  - **IPv4**: 32-bit address.
  - **IPv6**: 128-bit address.
- **Routing**: Directs packets between networks using routers.

---

## 5. Packet Structure
- **Packet**: A small chunk of data sent over a network.
  - **Header**: Contains source/destination addresses and other info.
  - **Payload**: The actual data being sent.

---

## 6. Error Handling and Flow Control
- **TCP**: Uses acknowledgments (ACK) to ensure data is received.
- **UDP**: No error handling; faster but less reliable.

---

## 7. Applications of TCP/IP
- **Web Browsing**: Uses HTTP/HTTPS over TCP.
- **Email**: Uses SMTP over TCP.
- **Streaming**: Uses UDP for faster delivery.
- **File Transfer**: Uses FTP over TCP.

---

## 8. Security in TCP/IP
- **HTTPS**: Secure version of HTTP using encryption (TLS/SSL).
- **IPSec**: Adds encryption and authentication to IP traffic.

---

## 9. Importance of TCP/IP
- **Backbone of the Internet**: Allows different devices to communicate.
- **Scalability**: Supports both small and large networks.
- **Interoperability**: Works across different types of networks and hardware.

---

## 10. Conclusion
- **TCP/IP**: The foundation of internet communication.
- **Flexible and Reliable**: Supports various protocols and use cases.

# Transmission Control Protocol (TCP) — Deep Dive

## 1. Why TCP Was Introduced

Before TCP, early ARPANET hosts used the **Network Control Program (NCP)** as the host-to-host protocol. NCP worked on a single packet-switched network but lacked mechanisms for:

- Interconnecting **multiple heterogeneous networks** (satellite, radio, LANs).
- **End-to-end reliability** across different networks.
- **Flow and congestion control** at scale.

As ARPANET grew and more networks were planned, NCP could not handle increasing traffic and heterogeneity. DARPA therefore funded the design of a new protocol suite that would:

- Allow **interconnection of multiple packet-switched networks** → a “network of networks”.
- Provide **reliable, ordered, full-duplex byte streams** between hosts.
- Be **host-based** rather than network-internal, so the core network could remain simple.

This led to the design of what we now call TCP/IP.

***

## 2. Origin: When and By Whom

- **Core paper:** “A Protocol for Packet Network Intercommunication”.
- **Authors:** Vinton G. Cerf and Robert E. Kahn (Vint Cerf & Bob Kahn).
- **Publication:** IEEE Transactions on Communications, **May 1974**.
- In that paper, TCP was originally conceived as a single **Transmission Control Program**, later split into:
  - **Internet Protocol (IP)** – best-effort datagram delivery.
  - **Transmission Control Protocol (TCP)** – reliable, ordered, congestion-controlled streams on top of IP.
- TCP/IP became operational in the ARPANET “flag day” cutover on **1 January 1983**.

***

## 3. Significance of TCP

### Foundational role

- TCP/IP is the **foundational protocol suite of the modern Internet**.
- TCP provides:
  - **Reliable delivery** (retransmissions, checksums, ACKs).
  - **Ordered delivery** (sequence numbers, reordering).
  - **Full-duplex byte streams** between processes.
  - **Flow control** (don’t overwhelm the receiver).
  - **Congestion control** (don’t melt the network).

### Why it mattered

- Enabled **interconnection of different networks** (ARPANET, SATNET, radio nets) into a single global Internet architecture.
- Shifted complexity to **end hosts**, keeping the network core simple, allowing the Internet to scale.
- Its congestion control evolution (e.g., Tahoe, Reno, NewReno, CUBIC) is a major reason the Internet survived explosive growth without repeated collapse.

### Today

- Still the default for Web (HTTP/1.1, HTTP/2), email (SMTP over TCP), SSH, FTP, and many RPC protocols.
- Newer transports (e.g., QUIC) explicitly reuse TCP’s **congestion control and reliability concepts**, migrating them to user space.

***

## 4. TCP Handshake and Algorithms

### 4.1 3-Way Handshake (Connection Establishment)

TCP uses a **3-way handshake** to establish a connection and agree on initial sequence numbers. From a **System Design perspective**, this introduces a **1-RTT (Round Trip Time) latency penalty** before any application data can be sent.

Let:
- Client initial sequence number (ISN_C) = `x`.
- Server initial sequence number (ISN_S) = `y`.

**Step 1 — SYN**
- Client → Server: segment with **SYN=1**, **ACK=0**, `Seq = x`.
- Meaning: “I want to start a connection, and my first byte will be numbered x.”

**Step 2 — SYN-ACK**
- Server → Client: segment with **SYN=1**, **ACK=1**, `Seq = y`, `Ack = x+1`.
- Meaning: “I acknowledge your SYN (next byte I expect is x+1) and here is my initial sequence y.”

**Step 3 — ACK**
- Client → Server: segment with **SYN=0**, **ACK=1**, `Seq = x+1`, `Ack = y+1`.
- Both sides transition to **ESTABLISHED** state and can start data transfer.

### 4.2 4-Step Teardown (Connection Termination)

Termination generally uses **FIN/ACK** exchanges (often called a 4-way handshake conceptually):

1. Side A sends **FIN** → “I’m done sending.”
2. Side B ACKs that FIN.
3. Side B later sends its own FIN.
4. Side A ACKs that FIN.

Both directions of the half-duplex stream close independently.

### 4.3 Reliability Algorithm (Simplified)

Core mechanisms:
- **Sequence numbers:** Every byte has a sequence number; segments carry `Seq` of first byte plus `Len`.
- **Acknowledgements (ACKs):** An ACK carries `Ack = next expected byte`; TCP uses **cumulative ACKs**.
- **Checksum:** Header + payload checksum detects corruption; corrupt segments are dropped and retransmitted.
- **Retransmission timers:** If an ACK isn’t received within RTO (Retransmission Timeout), sender retransmits.
- **Fast retransmit:** On receiving **3 duplicate ACKs** for the same `Ack`, sender retransmits **before** RTO fires.

### 4.4 Flow Control (Receiver-Side)

- Each TCP header contains a **Window** field advertising the receiver’s available buffer (receive window).
- Sender must ensure **bytes in-flight ≤ advertised window**.
- This prevents the sender from overwhelming a slow receiver.

### 4.5 Congestion Control (Network-Side)

TCP variants implement congestion control; classic ones:
- **Tahoe:** Slow start, congestion avoidance, fast retransmit.
- **Reno:** Adds fast recovery to better handle single losses.
- **NewReno:** Enhances Reno, better with multiple losses per window.

Core ideas:
- **Congestion window (cwnd):** Sender-maintained limit on outstanding unacked data.
- **Slow start:** Start with small cwnd and increase exponentially until a threshold.
- **AIMD (Additive Increase, Multiplicative Decrease):** Probes for bandwidth by adding 1 MSS per RTT; halves cwnd on loss.

***

## 5. Evolution of TCP Over the Years

### 5.1 High-Speed TCP Variants

As link speeds grew (high-bandwidth, long-RTT paths), classic Reno became too conservative.

- **TCP Vegas:** Measures RTT to estimate congestion before losses.
- **TCP BIC / CUBIC:** CUBIC is the default in Linux; uses a **cubic function** for cwnd growth, giving more aggressive window expansion.
- **BBR:** Google’s modern algorithm that models available bottleneck bandwidth and RTT instead of relying on packet loss signals.

### 5.2 TCP Options and Extensions

- **Window scaling:** Extend 16-bit window to support large windows for high-bandwidth paths.
- **Selective Acknowledgements (SACK):** Acknowledge non-contiguous ranges to recover from multiple losses efficiently.
- **Timestamps:** Better RTT measurement and PAWS.
- **MPTCP (Multipath TCP):** Use multiple paths (e.g., Wi-Fi + LTE) for a single connection.

***

## 6. Alternatives to TCP (System Design Trade-offs)

### 6.1 UDP (User Datagram Protocol)
- **Trade-offs:** Connectionless, best-effort; no handshake (0-RTT), no reliability, unordered. 
- **Header Size:** Lightweight 8-byte header (compared to TCP's 20-60 bytes).
- **Use Cases:** Low latency, real-time video, gaming, DNS.

### 6.2 SCTP (Stream Control Transmission Protocol)
- Message-oriented, multi-stream, multi-homing transport.
- Avoids Head-of-Line blocking by allowing multiple independent streams.

### 6.3 QUIC (IETF)
- UDP-based, user-space transport used by HTTP/3.
- Offers TLS 1.3 built-in, 0-RTT connection resumption, stream multiplexing without TCP’s head-of-line blocking.

***

## 7. Conceptual Deep Dive: From Basic to Expert Level

### 7.1 Overhead & Statefulness

- **Memory Overhead:** TCP is a **stateful** protocol. Both the client and server must allocate OS-level memory to track sequence numbers, congestion windows, and packet buffers. In massive distributed systems, millions of idle TCP connections can exhaust server memory.
- **Header Overhead:** The TCP header is a minimum of 20 bytes, making it heavier on the wire than UDP.

### 7.2 Connection Lifecycle & States

TCP is fundamentally a **state machine**. Key states:
- **LISTEN:** Server waiting for connections.
- **ESTABLISHED:** Connection up, full-duplex data flow.
- **TIME-WAIT** (typically 2×MSL, e.g., 2 minutes): Endpoint that sends the last ACK stays in TIME-WAIT to catch delayed segments and prevent old duplicates from being misinterpreted as new data. High volumes of short-lived connections can exhaust ephemeral ports due to TIME-WAIT accumulation.

### 7.3 Advanced Topics for Engineering

#### Head-of-Line (HoL) Blocking
- TCP delivers a **single ordered byte stream**.
- If packet with seq `x` is lost but packets `x+1, x+2, …` arrive, the receiver must **buffer** them but cannot deliver them to the application until `x` is retransmitted. This forces modern protocols (like HTTP/3) to move to UDP.

#### Nagle’s Algorithm and Delayed ACKs
- **Nagle’s algorithm:** Coalesces small writes into larger segments to reduce tiny packets.
- **Delayed ACKs:** Receiver delays ACK to piggyback on data.
- **Interaction:** Together, these can cause severe latency spikes. High-performance latency-sensitive systems (like gaming or real-time trading) disable Nagle via `TCP_NODELAY`.

#### Load Balancers and TCP
- Stateful L4 load balancers and NAT devices must track **TCP state** to route packets correctly. 
- If a connection is idle too long, the load balancer may silently drop the state. Applications must implement application-level keep-alives (ping/pong) to prevent this.
