Below is a single, self‑contained Markdown “note” you can save as  
`akamai-globally-distributed-cdn.md`. It is grounded entirely in the attached paper by Dilley, Maggs, Parikh, Prokop, Sitaraman, and Weihl (IEEE Internet Computing, 2002). 

***

# Akamai’s Globally Distributed Content Delivery Network  
*(Based on “Globally Distributed Content Delivery” – Akamai Technologies)*

> Paper: John Dilley, Bruce Maggs, Jay Parikh, Harald Prokop, Ramesh Sitaraman, Bill Weihl – *IEEE Internet Computing*, Sept/Oct 2002. 

***

## 1. Motivation: The “Flash Crowd” Problem

### 1.1 Flash crowds and bottlenecks

As web sites grow popular, a sudden spike in traffic (a **flash crowd**) can overwhelm parts of the site’s infrastructure. 

Typical bottlenecks:

- **Front‑end web servers**  
- **Network equipment / bandwidth** to the data center  
- **Back‑end transaction infrastructure** (databases, application servers) 

Consequences:

- Crashes and very high response times.  
- Direct business impact: lost revenue and negative user perception. 

Akamai evolved from an MIT research effort specifically aimed at solving this flash‑crowd problem by distributing content delivery across the edge of the network. 

### 1.2 Why “single location” serving is fragile

Serving all content from a single data center causes issues for:

- **Scalability** – need enough capacity for peak (often 10× average) load.  
- **Reliability** – data center or ISP failure makes the site unreachable.  
- **Performance** – users far from the DC suffer from high latency and intermediate congestion. 

Even with clustering, mirroring, and multihoming (multiple ISPs), you need significant **excess capacity** at each site/connection to survive peaks and failures, which becomes expensive. 

***

## 2. Existing Approaches Before Akamai

The paper surveys three main pre‑CDN approaches. 

### 2.1 Clustering

- Multiple servers behind a single front door (load balancer).  
- Pros: handles local failures, scales within a data center.  
- Cons:
  - If the **data center or ISP fails**, the whole cluster disappears.  
  - Hard to scale to thousands of servers at Internet‑wide scale. 

### 2.2 Mirroring

- Deploy identical clusters in a few geographically separated locations.  
- Clients are sent to one of the mirrors.  
- Cons:
  - Must **synchronize** content between mirrors (hard, especially for dynamic content).  
  - Each mirror must be sized to handle **full peak load**. 

### 2.3 Multihoming

- Connect a site to multiple ISPs.  
- Pros: partial resilience to ISP failures and “first‑mile” problems.  
- Cons:
  - Underlying BGP (Border Gateway Protocol) doesn’t always converge quickly on new routes when connections fail.  
  - Each upstream link must be able to handle **entire traffic** in failure mode. 

All three approaches require substantial **over‑provisioning** (servers and bandwidth) and still do not address all network‑level congestion and failure points (peering points, backbones, last‑mile). 

***

## 3. Akamai’s Basic Idea: Surrogate Servers at the Edge

### 3.1 Surrogates and caching at the edge

Akamai’s system serves web content from a **variable number of surrogate origin servers at the Internet’s edge**. 

Core idea:

- Place many servers inside **thousands of networks**, close to end users.  
- **Cache content** at those edge servers to:
  - Reduce load on origin infrastructure.  
  - Shorten network distance (lower latency).  
  - Avoid backbone and peering point congestion. 

By early 2000s:

- The system had **> 12,000 servers across > 1,000 networks** worldwide. 

### 3.2 Why ISP proxy caches weren’t enough

ISPs and enterprises had deployed **proxy caches** to reduce latency and bandwidth. 

Limitations:

- Cache hit rates typically **25–40%**:
  - Increasing use of dynamic content.  
  - Many objects either uncacheable or short‑lived. 
- Caches weren’t coordinated with content providers:
  - Little support for invalidation, authorization, dynamic assembly.  
  - No incentive for sites to adapt to arbitrary cache implementations. 

Akamai’s advantage:

- It **controls both network and software**, plus works directly with content providers:
  - Can add features like explicit invalidation, authorization, dynamic assembly (ESI).
  - Can make “otherwise uncacheable” content cacheable at the edge. 

***

## 4. Akamai’s Network Infrastructure

### 4.1 Goals

The infrastructure must:

- Handle **flash crowds** by allocating more servers to hot sites.  
- Always serve users from **nearby servers**.  
- React quickly to **server failures** and **network conditions**.  
- Scale to **tens of thousands of servers** across many networks. 

### 4.2 Key concept: “Nearest Available Likely” server

Akamai directs each client request to a server that is:

- **Nearest** – in terms of network metrics:
  - Lower **round-trip time (RTT)**.  
  - Lower **packet loss**.  
- **Available** – with respect to:
  - Server load (CPU, disk, network).  
  - Data center bandwidth capacity.  
- **Likely** – likely to have the requested content:
  - Based on which servers at a site store which customers’ objects (via **consistent hashing**). 

If any dimension is bad (e.g., high load, network congestion, server down), that server is avoided.

***

## 5. Mapping and DNS‑Based Request Routing

### 5.1 DNS‑driven mapping

Akamai uses a **dynamic, fault‑tolerant DNS system** to translate hostnames into IPs of edge servers. 

Mapping process:

1. Client resolves an Akamai hostname (e.g., `a7.g.akamai.net`) using standard DNS.  
2. Akamai’s **top‑level DNS (TLD NS)** and **low‑level DNS** participate in resolution.  
3. Mapping logic decides which set of edge servers (IP addresses) to return, based on:
   - Service type, server health, load, network conditions, client location, and content distribution. 

### 5.2 DNS resolution steps (from the DNS Resolution sidebar)

A typical resolution proceeds like this: 

1. **Root query**  
   Resolver asks a root name server for `a7.g.akamai.net`.  
   Root responds with NS records for `.net`.

2. **.net query**  
   Resolver asks `.net` servers for `a7.g.akamai.net`.  
   They return delegation (NS) for `.akamai.net` (Akamai top‑level DNS).

3. **Akamai top‑level DNS**  
   Resolver queries Akamai TLD NS, which returns delegation for `.g.akamai.net` to some **low‑level Akamai name servers**, with TTL ≈ 1 hour.  
   The selected low‑level NS are **co‑located** with candidate edge servers near the user. 

4. **Akamai low‑level DNS**  
   Resolver queries the low‑level NS, which returns **actual edge server IP addresses** for that hostname, with a **short TTL** (several seconds to about a minute). 

Short TTLs allow Akamai to:

- Adapt quickly to load changes, failures, and network shifts.  
- Re‑map clients to different edge servers or data centers as conditions change. 

Caching behavior:

- If the resolver has cached `g.akamai.net` NS, it can skip some steps.  
- Standard DNS caching rules apply; Akamai tunes TTLs to balance overhead and agility. 

### 5.3 BGP and network measurement

To define “nearest” and understand topology:

- Akamai agents peer with certain **border routers** and receive **BGP** updates (autonomous system paths). 
- BGP gives a coarse notion of network distance (AS hops).  
- Akamai combines BGP info with active measurements like **traceroute** to build a **dynamic map of Internet structure and link quality**. 

This dynamic map feeds the mapping system, which updates mappings every few seconds while avoiding excessive DNS overhead. 

***

## 6. Monitoring and Automatic Network Control

### 6.1 Server and data-center load reporting

Each content server for HTTP, HTTPS, and streaming:

- Periodically reports its **load** to a monitoring application:
  - CPU, disk, network usage.  
- The monitored data is aggregated and published to local DNS servers. 

DNS servers then use this information to:

- Decide which IP addresses to return for a hostname.  
- If a server’s load crosses thresholds:
  - Some of its assigned content is reallocated to other servers.  
  - Eventually, its IP may be removed from DNS answers so it can “shed load”.

Data center load metrics are propagated upward to **top‑level DNS** so mappings can avoid overloaded data centers. 

### 6.2 End‑to‑end health monitoring

Akamai also uses **agents that simulate end‑user behavior**:

- Download web objects.  
- Measure failure rates and download times.  
- Detect problematic data centers or servers. 

Use cases:

- Automatically suspend problematic data centers.  
- Provide real‑time application metrics and operational visibility.  
- Feed into the “customer traffic analyzer” for network ops and billing.

***

## 7. Network Services Built on the CDN

The paper describes how the same infrastructure supports multiple services. 

### 7.1 Static content

Static content includes:

- HTML, images, executables, PDFs, etc.  

Akamai applies **content‑type‑specific lifetimes and features**:

- **Lifetime (TTL) per object**, ranging from:
  - 0 seconds (validate on every request with origin).  
  - Infinite (never revalidate unless explicitly purged). 
- Edge server lifetime can differ from downstream proxies and clients.  
- Special features:
  - HTTPS support, alternate encodings, cookies, etc.  
  - Controlled via a **metadata facility** that decides which features apply per customer and content type. 

### 7.2 Dynamic content and ESI (Edge Side Includes)

Problem:

- Traditional proxy caches can’t easily handle dynamic pages (e.g., user‑personalized pages, frequently changing ads). 

Solution: **ESI – Edge Side Includes** 

- ESI is like server‑side includes but optimized for **edge assembly** and fault tolerance.  
- Pages are divided into **fragments** with independent cacheability:
  - Static fragments may be cached for long.  
  - Dynamic fragments may be fetched or computed per request. 
- Fragments are stored as separate cache objects and **assembled into a page at the edge**.  
- ESI integrates an **XSLT engine** for processing XML data. 

Benefits:

- Only the **non‑cacheable or expired fragments** need to be fetched from origin.  
- Dramatically reduces origin bandwidth and load for dynamic sites.  
- In studied sites (portals, finance), ESI reduced bandwidth requirements for dynamic content by **95–99%** across the WAN. 

### 7.3 Streaming media

Akamai supports live and on‑demand streaming for:

- Microsoft Windows Media  
- RealNetworks’ formats  
- QuickTime 

Key challenges:

- **Live streams**:
  - Origin captures and encodes the stream and sends it to an **entry‑point server** in Akamai’s network.  
  - Must quickly failover to another entry point if one fails (no interruption).  
  - Streams must be pushed from entry point to edge servers with:
    - Resilience to network failures/packet loss.  
    - Low delay and jitter (late/out‑of‑order packets are useless). 

- Techniques:
  - **Information dispersal**: send data along multiple redundant paths from entry point to edge; edge can reconstruct stream even if some paths are down or lossy. 

- **On‑demand**:
  - Provider uploads clips into a distributed storage facility, replicated across many DCs.  
  - On request, the edge server:
    - Fetches content from the optimal storage location.  
    - Caches it locally while serving the stream.

***

## 8. System‑Level Challenges and Solutions

The paper devotes a large section to operational challenges. 

### 8.1 Scalability

Challenges:

- Monitoring and controlling **tens of thousands of servers** with minimal overhead.  
- Monitoring network conditions between **thousands of locations** and updating mappings **every few seconds**.  
- Dealing with **incomplete, out‑of‑date information** about network and load.  
- Reacting quickly to changing network conditions and workloads.  
- Measuring Internet conditions at fine enough granularity to estimate end‑user performance.  
- Managing many customers with different:
  - Workloads  
  - Content sizes  
  - Service needs  
- Ensuring **customer isolation** (one customer’s spike should not affect others).  
- Maintaining **data integrity** across many TB of storage (end‑to‑end checks beyond local disk checks).  
- Collecting and processing **massive logs** for billing and analytics. 

Solutions mentioned:

- **Distributed monitoring service** resilient to temporary info loss.  
- Frequent mapping recomputations, but with careful optimization to avoid DNS lookup latency.  
- End‑to‑end integrity checks for cached objects.  
- Sophisticated logging and back‑end processing pipelines. 

### 8.2 Reliability

Sources of failure:

- Hardware aging (especially disks).  
- Network problems (routers, switches).  
- Software bugs and protocol changes (new headers in browsers, servers). [ppl-ai-file-upload.s3.amazonaws]

Approaches:

- **Massive replication**:
  - Many edge servers per site.  
  - Multiple DNS servers; top‑level DNS ensures clients can find a live DNS server.  
- DNS returns **multiple IP addresses** for a name:
  - Clients can try another IP if the first fails.  
  - Live servers at a site can adopt the IP of a failed server. [ppl-ai-file-upload.s3.amazonaws]
- Avoid single points of failure:
  - Replicate monitoring and control logic.  
  - Multiple DNS sites and TLD/low‑level hierarchies. [ppl-ai-file-upload.s3.amazonaws]
- Testing new software:
  - Special test tools mirror live traffic to a **test version** of software (shadow traffic).  
  - Find issues before rolling to production. [ppl-ai-file-upload.s3.amazonaws]

### 8.3 Software deployment and platform management

Reality:

- You cannot update all edge servers atomically.  
- Different servers/networks may be unreachable at different times.  
- Often **two software versions coexist** live. [ppl-ai-file-upload.s3.amazonaws]

Implications:

- Components must be designed so that **different versions interoperate** safely.  
- Network ops must identify and suspend misconfigured servers.  
- Akamai runs both **Linux and Windows** on edge servers; tools must work across platforms. [ppl-ai-file-upload.s3.amazonaws]

### 8.4 Content visibility and control

Key requirements:

- Providers must control:
  - Cache lifetimes.  
  - Cache consistency / invalidation.  
  - Authentication and authorization.  
  - Content integrity.  
- Providers need visibility:
  - Detailed logs and statistics.  
  - Real‑time and historical views of traffic. [ppl-ai-file-upload.s3.amazonaws]

Mechanisms:

1. **Cache consistency & lifetime**
   - TTL policies:
     - Some objects cache forever (until explicitly purged).  
     - Others have finite TTL; edge revalidates or refetches.  
   - Versioned URLs:
     - Embedding version/generation in URL or query string; new version → new URL; older versions cached long‑term. [ppl-ai-file-upload.s3.amazonaws]

2. **Performance for uncacheable content**
   - Edge servers sit between client and origin, **splitting TCP connections**:
     - Edge can react quickly to packet loss, improving effective throughput.  
     - Edge can absorb origin’s response quickly and stream it to client at client pace.  
     - Edge keeps long‑lived connections to client; origin maintains fewer connections to fewer edge servers. [ppl-ai-file-upload.s3.amazonaws]

3. **On‑demand purge (lifetime control)**
   - Edge servers support **on‑demand invalidation** of objects across the network:
     - Via customer requests or publishing system integration.  
   - Essential because most objects change infrequently, but some must be removed immediately upon update or policy changes. [ppl-ai-file-upload.s3.amazonaws]

4. **Authentication and authorization**
   - Two models:
     - Edge passes request headers to origin so origin can authorize each request.  
     - Edge processes **authorization tokens** issued by origin (no round trip for every request).  
   - Edge ensures that failed authorization does **not** purge or corrupt stored protected content. [ppl-ai-file-upload.s3.amazonaws]

5. **Integrity control**
   - Edge must:
     - Serve correct responses (no mix‑up between customers).  
     - Detect incomplete or corrupted origin responses and avoid caching them.  
     - Detect disk corruption of cached objects and refetch.  
   - Akamai adds **content integrity checks** before serving each block of a response to ensure correct association between request and bytes. [ppl-ai-file-upload.s3.amazonaws]

6. **Visibility (logs and billing)**
   - Edge servers log each request.  
   - Logs are aggregated:
     - For customer‑specific reporting.  
     - For billing (large volume of data, must be reduced to monthly summaries).  
   - Real‑time analytics focus on rates and geographic distribution rather than full detailed logs. [ppl-ai-file-upload.s3.amazonaws]

***

## 9. Relation to Other Distributed/Fault‑Tolerant Systems

The authors explicitly connect Akamai’s design to broader distributed systems research. [ppl-ai-file-upload.s3.amazonaws]

Key points:

- Many Internet subsystems are **decentralized** (routing, DNS, email, web).  
- Akamai uses a **logically centralized but physically distributed** control layer:
  - Central logic for mapping and management.  
  - Replicated control components and edge nodes. [ppl-ai-file-upload.s3.amazonaws]

References in the side‑bar:

- **Autonet** – example of centralized route recomputation on topology changes. [ppl-ai-file-upload.s3.amazonaws]
- Classic work on **distributed databases**, **distributed file systems**, and **caches**.  
- Web caches historically had limited impact due to:
  - Mostly read‑only web but high rate of change.  
  - Lack of coordination between caches and data sources. [ppl-ai-file-upload.s3.amazonaws]
- Depot and other update‑management tools for large distributed software deployments; Akamai takes similar goals but uses its **own network plus public‑key mechanisms** for software distribution. [ppl-ai-file-upload.s3.amazonaws]

***

## 10. Edge Computing: Future Direction

The paper ends by foreshadowing **edge applications** beyond content delivery. [ppl-ai-file-upload.s3.amazonaws]

Key ideas:

- Running applications at the edge has similar benefits to content delivery:
  - Capacity on demand.  
  - Shared infrastructure.  
  - Reduced latency (less long‑distance communication). [ppl-ai-file-upload.s3.amazonaws]
- New challenges:
  - Visibility into application behavior when app instances are moving between machines.  
  - Sandbox isolation to prevent apps from interfering.  
  - Distributed data for these apps (not just static content).  
  - Identifying **practical design patterns** rather than solving the “general case” (which is often impossible). [ppl-ai-file-upload.s3.amazonaws]

This is essentially an early articulation of what we now call **edge computing / serverless at the edge**.

***

## Abbreviations Used

- **AS** – Autonomous System (routing domain used by BGP)  
- **BGP** – Border Gateway Protocol  
- **CDN** – Content Delivery Network  
- **DC** – Data Center  
- **DNS** – Domain Name System  
- **ESI** – Edge Side Includes  
- **HTTP/HTTPS** – Hypertext Transfer Protocol (Secure)  
- **IXFR** – Incremental Zone Transfer (from DNS context; referenced in other work, not core here)  
- **RTT** – Round‑Trip Time  
- **TCP** – Transmission Control Protocol  
- **TTL** – Time To Live (caching or DNS resolution lifetime)  

***

## References

- J. Dilley, B. Maggs, J. Parikh, H. Prokop, R. Sitaraman, B. Weihl, “Globally Distributed Content Delivery,” *IEEE Internet Computing*, vol. 6, no. 5, Sept./Oct. 2002.  
  Public copies and metadata:  
  https://www.cs.unc.edu/~jasleen/Courses/COMP631/papers/AkamaiOverview.pdf  
  https://www.semanticscholar.org/paper/Globally-Distributed-Content-Delivery-Dilley-Maggs/e381902436a736cc170e15c9cfc938748470bcd0  

- Y. Rekhter, T. Li, “A Border Gateway Protocol 4 (BGP‑4),” RFC 1771, Mar. 1995.  
  RFC Editor entry: https://www.rfc-editor.org/info/rfc1771  
  BGP‑4 overview: https://www.freesoft.org/CIE/Topics/88.htm  

- G. Malkin, “Traceroute Using an IP Option,” RFC 1393, Jan. 1993.  
  (Obsoleted, but still useful for historical context.)  
  Brief description: http://www.punto-bit.com/cgi-bin/main.pl?sh=1208&la=en  

- D. Karger et al., “Consistent hashing and random trees: distributed caching protocols for relieving hot spots on the World Wide Web,” *STOC* 1997.  
  Citation and metadata:  
  https://www.semanticscholar.org/paper/Consistent-hashing-and-random-trees%3A-distributed-on-Karger-Lehman/02bb762c3bd1b3d1ad7883402c5a34b7a2a0b4c2  
  Introductory notes on consistent hashing:  
  https://web.stanford.edu/class/cs168/l/l1.pdf  

- Additional background on Akamai’s platform (post‑2002):  
  https://dl.acm.org/doi/10.1145/1842733.1842736
***

To deepen your understanding: based on this architecture, how would you explain the difference between **Akamai’s DNS‑based mapping** and a simple **round‑robin DNS** load balancing setup in terms of failure handling and performance?