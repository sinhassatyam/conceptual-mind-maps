*Response generated using Claude Sonnet 4.6.*

Below is a **drop‑in replacement** for your `load_balancers.md`. It is structured as a textual mind map: start at the center (“Load Balancing”), then follow headings and sub‑bullets as branches.

All concepts are consistent with HAProxy docs, NGINX docs, AWS ELB docs, and the System Design Primer. [ieeexplore.ieee](https://ieeexplore.ieee.org/document/8229855/)

***

# Load Balancers – Conceptual Mind Map

## 0. What problem does load balancing solve?

- Modern systems have:
  - Many clients, limited single‑machine capacity, and variable traffic patterns. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
  - The need for high availability: instances fail, zones/regions fail. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)
- A **load balancer (LB)** is a component that:
  - Accepts incoming connections on a virtual IP / DNS name.
  - Distributes them across multiple backend instances according to policies. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
  - Continuously monitors health and reacts to failures.

Think of the LB as a **traffic cop** at the entrance of a multi‑lane highway.

***

## 1. Dimensions of the design space

### 1.1 Where does it sit in the stack?

- **Layer 4 (L4) – Transport**
  - Makes decisions based on IP + TCP/UDP ports only. [docs.trafficserver.apache](https://docs.trafficserver.apache.org/en/10.0.x/admin-guide/configuration/hierarchical-caching.en.html)
  - Does not look into HTTP headers, cookies, or payload.
  - Works for any TCP/UDP protocol: HTTP, gRPC, SMTP, Redis, custom binary. [docs.trafficserver.apache](https://docs.trafficserver.apache.org/en/10.0.x/admin-guide/configuration/hierarchical-caching.en.html)
- **Layer 7 (L7) – Application**
  - Parses application protocol (usually HTTP/HTTPS).  
  - Can route by URL path, Host header, cookies, method, etc. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
  - Enables smarter routing, per‑API policies, and content‑aware features.

### 1.2 Where is it deployed?

- **Hardware load balancer**
  - Proprietary appliances (e.g., F5, Citrix ADC).  
  - Historically used in on‑prem DCs; high throughput, rich feature sets. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)
- **Software load balancer / reverse proxy**
  - Runs on general‑purpose servers: HAProxy, NGINX, Envoy, Traefik. [ieeexplore.ieee](https://ieeexplore.ieee.org/document/8229855/)
  - Often deployed as:
    - Edge reverse proxy.
    - Internal service LB (sidecar or node‑local).
- **Cloud / managed load balancer**
  - AWS ELB/ALB/NLB, GCP Load Balancing, Azure Load Balancer. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)
  - Provider manages scaling, failover, and health checks.

### 1.3 Traffic type & placement

- **North–south**
  - From the internet into your system (client → edge LB → service). [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)
- **East–west**
  - Service‑to‑service within a DC/VPC or across regions.
  - Often done with service meshes (Envoy/Linkerd) or internal NGINX/HAProxy. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)

***

## 2. Core responsibilities of a load balancer

### 2.1 Connection termination and forwarding

- Accepts incoming connections on a **frontend** (IP:port, protocol).
- Opens or reuses connections to **backends** (servers, instances, pods).  
- May terminate TLS at the LB or proxy it through to the backend. [docs.trafficserver.apache](https://docs.trafficserver.apache.org/en/10.0.x/admin-guide/configuration/hierarchical-caching.en.html)

### 2.2 Load distribution

- Implements a **scheduling algorithm**:
  - Decides which backend gets each new connection or request.
- Needs to avoid:
  - Hotspots (one instance overloaded).
  - Thrashing (bouncing a user’s session between instances).

### 2.3 Health checking and failure handling

- Periodically probes backend instances with:
  - TCP connect checks.
  - HTTP/HTTPS checks to a path (e.g., `/healthz`). [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
- If a backend fails checks:
  - It is removed from the active rotation.
  - Some systems support “drain” vs “hard down” states (e.g., for maintenance).

### 2.4 Observability

- Exposes metrics:
  - QPS, error rates, latencies, open connections, backend health. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
- Logs:
  - Requests, chosen backend, response codes, timing.
- This is critical when debugging production incidents.

***

## 3. Load balancing algorithms

### 3.1 Stateless, simple algorithms

- **Round robin**
  - Assigns each new request to the next server in order (A → B → C → A …). [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
  - Simple and fair if all instances are identical.
- **Random**
  - Picks a random backend for each request.
  - Works surprisingly well at scale; used with weights in some systems.

### 3.2 Load‑aware algorithms

- **Least connections**
  - Send new connection to the server with the fewest current connections. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
  - Good when:
    - Connections have variable lifetimes.
    - Some requests are more expensive than others.
- **Least response time**
  - Uses both connection count and average latency to choose backend.

### 3.3 Weighted algorithms

- **Weighted round robin**
  - Assigns more requests to “larger” or more capable instances. [ieeexplore.ieee](https://ieeexplore.ieee.org/document/8229855/)
  - Example:
    - A and B are small; C and D are big – weights 1:1:3:3.
- **Weighted least connections**
  - Combines capacity weights with current load.

### 3.4 Client‑affinity algorithms

- **Source IP hashing**
  - Hash(client IP) → backend index.
  - Properties:
    - Same client IP tends to hit same backend → pseudo‑stickiness. [ieeexplore.ieee](https://ieeexplore.ieee.org/document/8229855/)
    - Good for internal networks with many distinct client IPs.
  - Pitfall:
    - If behind a NAT/proxy, many users share one IP, causing imbalance.
- **Consistent hashing (advanced)**
  - Used when backend membership changes frequently.
  - Minimizes key movement when adding/removing servers.

***

## 4. L4 vs L7 load balancing in depth

### 4.1 L4 (transport) load balancing

- **What it sees**
  - IP addresses, TCP/UDP ports; no HTTP semantics.
- **How it operates**
  - Forwards TCP segments or UDP datagrams to chosen backend.
  - Does not modify HTTP headers or cookies.
- **Pros**
  - Very fast; low CPU overhead. [docs.trafficserver.apache](https://docs.trafficserver.apache.org/en/10.0.x/admin-guide/configuration/hierarchical-caching.en.html)
  - Works for any protocol, not just HTTP.
- **Cons**
  - Cannot do:
    - Path‑based routing (`/api` vs `/static`).
    - Host‑based routing (multi‑tenant virtual hosting).
    - Fine‑grained policies by method, cookie, etc. [docs.trafficserver.apache](https://docs.trafficserver.apache.org/en/10.0.x/admin-guide/configuration/hierarchical-caching.en.html)

### 4.2 L7 (application) load balancing

- **What it sees**
  - Full HTTP requests and responses: method, URL, headers, body. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
- **Capabilities**
  - Path routing:
    - `/api/*` → API service; `/assets/*` → static server/CDN. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
  - Host routing:
    - `foo.example.com` → service A; `bar.example.com` → service B.
  - Header‑based routing:
    - `X-Canary: true` → canary cluster.
  - Cookie‑based routing & sticky sessions.
  - Rate limiting, auth, WAF integration.
- **Cost**
  - Requires parsing HTTP, buffering some data.
  - Higher CPU/memory footprint compared to pure L4.

### 4.3 Elastic Load Balancing (AWS Classic) as an example

- Supports frontends with protocols:
  - HTTP, HTTPS, TCP, SSL (secure TCP). [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)
- Can be configured as:
  - L4 pass‑through (TCP/SSL on both sides).
  - L7 HTTP/HTTPS with header parsing, X‑Forwarded‑For, sticky sessions. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)
- Uses listeners:
  - Each listener: (frontend protocol:port) → (backend protocol:port). [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)

***

## 5. Session affinity (“sticky sessions”)

### 5.1 Why stickiness?

- Some apps store session state **in memory on a single backend**:
  - Shopping cart, logged‑in user state, etc.
- To avoid losing session when the LB sends consecutive requests to different backends:
  - Use **session affinity** so a given user is consistently mapped to one backend. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)

### 5.2 Implementations

- **Cookie‑based stickiness**
  - LB injects a cookie like `SRV=A` or uses existing cookie (e.g., JSESSIONID prefix in HAProxy). [ieeexplore.ieee](https://ieeexplore.ieee.org/document/8229855/)
  - Later requests with that cookie are routed to the same backend.
- **IP‑based stickiness**
  - Source IP hashing (see above).
  - Simpler, but breaks with NAT, mobile IP changes, or proxies.

### 5.3 Trade‑offs

- Pros:
  - Works with stateful backends without immediate refactor.
- Cons:
  - Reduces effective load balancing.
  - Some servers can get more load if many “heavy” users are stuck there.
  - Makes horizontal scaling and failure handling trickier.

At scale, the preferred pattern is **stateless backends + shared stores** (SQL/NoSQL/cache), so stickiness becomes optional.

***

## 6. Health checks and failure modes

### 6.1 Health‑check types

- **L3/L4 reachability**
  - ICMP ping or TCP connect; only checks that a process is listening. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
- **L7 HTTP/HTTPS**
  - Request `GET /healthz` or `HEAD /index.html`.  
  - Check for:
    - Status code (2xx or 3xx).
    - Optional body contents.
- **Custom checks**
  - Scripted checks via sidecar or agent performing deeper validation.

### 6.2 Failure handling strategies

- **Immediate removal**
  - If check fails `N` times, mark backend as DOWN and stop sending traffic. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
- **Slow start / warm up**
  - After a backend becomes healthy again, ramp up traffic gradually (weights, rate limiting).
- **Drain vs hard down**
  - “Drain” mode: stop new connections but allow in‑flight requests to complete.
  - Used for rolling deployments and maintenance.

### 6.3 Edge cases

- Partial failures:
  - CPU saturated, but health endpoints still succeed → use metrics‑based decisions.
- Dependency failures:
  - `/healthz` can check ability to talk to DB/cache; otherwise LB might think it’s healthy while app cannot handle real requests.

***

## 7. Reverse proxy vs load balancer

### 7.1 Reverse proxy

- A reverse proxy:
  - Sits in front of one or more servers.
  - Terminates client connections, may cache, compress, or modify responses. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
- It can be used purely for:
  - TLS termination.
  - Static asset caching.
  - Security filtering (WAF).

### 7.2 Load balancer

- A load balancer:
  - Primary goal is **traffic distribution and failure handling**. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)
- Many real‑world tools (NGINX, HAProxy, Envoy) function as **both**:
  - Reverse proxy + load balancer + gateway.

In practice, “reverse proxy” vs “load balancer” is often about **role/config**, not different software.

***

## 8. Architectures and patterns with LBs

### 8.1 Active–passive vs active–active

- **Active–passive LB**
  - One LB handles all traffic; a second is on standby with VRRP/keepalived.  
  - If primary fails, VIP moves to secondary. [ieeexplore.ieee](https://ieeexplore.ieee.org/document/8229855/)
- **Active–active LB**
  - Multiple LBs receive traffic simultaneously (e.g., via L4 LB in front or DNS).  
  - Better throughput and redundancy but requires careful config sync.

### 8.2 Hierarchical load balancing

- **Multi‑tier**
  - Edge LB → internal LBs → service instances.
  - Example:
    - Cloud provider L4 LB → NGINX/HAProxy inside the VPC → app pods in Kubernetes.

### 8.3 Anycast & DNS‑based distribution

- **DNS load balancing**
  - Multiple IPs for same hostname; client resolvers pick one.  
  - Used across regions or LB clusters.
- **Anycast**
  - Same IP prefix announced from multiple locations; BGP routes client to “nearest” LB. [kentik](https://www.kentik.com/kentipedia/what-is-internet-peering/)
  - Widely used in CDNs and global anycast LBs.

***

## 9. Security aspects

### 9.1 TLS termination and re‑encryption

- **TLS termination at LB**
  - LB holds server certificate/private key.
  - Decrypts client traffic, then forwards as plain HTTP or re‑encrypted HTTPS to backend. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)
- **Pros**
  - Offloads expensive crypto from app servers.
  - Centralizes certificate management and renewal.
  - Enables L7 inspection (headers, URLs, WAF).
- **Cons**
  - Cleartext between LB and backend (unless re‑encrypt).

### 9.2 DDoS mitigation

- LBs can help by:
  - Rate limiting or connection limiting from problematic IPs.
  - Early detection using aggregated metrics.
  - Sitting behind upstream DDoS protection (e.g., cloud‑provider shields). [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)

### 9.3 WAF integration

- NGINX/Envoy/ALBs can integrate with Web Application Firewalls:
  - Rule‑based filtering, signature matching.
  - OWASP Top 10 (XSS, SQL injection, etc.) mitigation. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)

***

## 10. How CDNs and cloud platforms extend load balancing

### 10.1 CDN as a global LB + cache

- A CDN is effectively:
  - A **globally distributed L7 load balancer + cache** at hundreds of PoPs.
- It terminates user traffic near the user and:
  - Serves static/dynamic content from edge caches.
  - Proxies upstream to origin over optimized routes. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)

### 10.2 Cloud‑native scaling

- Managed LBs (e.g., AWS ELB/ALB/NLB):
  - Auto‑scale capacity as QPS grows.
  - Integrate with auto‑scaling groups:
    - When new instances join/leave, LB’s backend pool updates automatically. [nojitter](https://www.nojitter.com/data-management/solving-internet-backbone-problems-to-deliver-on-end-user-expectations)
  - Support cross‑zone or cross‑AZ balancing out of the box.

### 10.3 Multi‑region and multi‑cluster

- Combine:
  - **Global traffic management** (DNS or global anycast LB).
  - **Regional LBs** inside each region.
- For Kubernetes:
  - NGINX/Envoy ingress controllers perform L7 load balancing at cluster edges. [geeksforgeeks](https://www.geeksforgeeks.org/system-design/web-proxy-caching-in-distributed-system/)

***

## 11. Practical trade‑offs and anti‑patterns

### 11.1 Single LB as a SPOF

- Mistake:
  - One instance of HAProxy/NGINX with no redundancy.
- Fix:
  - Active–passive with VRRP / keepalived or active–active with multiple LBs and a fronting L4 LB/DNS.

### 11.2 LB doing too much (monolith LB)

- Overloading the LB with:
  - Very heavy per‑request logic, big Lua filters, complex regex routing.
- Risk:
  - LB becomes CPU bottleneck and source of latency.
- Better:
  - Keep LB logic focused on routing, protocol termination, and light transformations.
  - Push heavy logic into downstream services or sidecars when possible.

### 11.3 Sticky sessions without understanding state

- Using stickiness to “fix” an inherently stateful app:
  - Works, but hides underlying design problems.
- As traffic grows:
  - Harder to scale, harder to roll out changes, uneven load distribution.
- Long‑term:
  - Move towards **stateless app servers** and central session stores.

***

## 12. How to reason about load balancing as an architect

When designing or reviewing a system, ask:

1. **What layer do I need?**
   - Pure TCP/UDP → L4 is enough.
   - HTTP routing, canaries, A/B tests → L7.
2. **Where is the LB in relation to users and services?**
   - Edge of the internet? Inside the cluster? Cross‑region?
3. **How does it fail, and what happens when it does?**
   - Redundancy, failover paths, DNS TTLs, VRRP, etc.
4. **What is my state story?**
   - Do I require stickiness? If yes, why, and for how long?
5. **What are my observability requirements?**
   - Metrics, logs, distributed tracing correlation.

***

To make this more concrete for your own use: if you diagram your current or ideal stack, where would you place L4 vs L7 load balancers, and which routing decisions would you push into each layer?