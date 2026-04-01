# Systems Design for Interviews – Mind Map

## 1. Networks and Protocols
  - **Networks**: Connections
  - **Protocols**: rules that govern something, official procedure.
- **IP (Internet Protocol)**
  - Protocol/rules that govern the Internet - Internet Protocol
  - IP Address is a numeric label assigned to devices connected over networks using Internet Protocol for communication.
  - Messages sent in **packets**
    - **Definition of Packets**
      - small chunk of information (2^16 bytes)
    - **Components**
      - headers (source & destination IP addresses)
      - data (message/information)
  - **IPv4** (2^32) vs. **IPv6** (2^128 to handle IP exhaustion).
  - **Demerits (Corrupted Data)**
    - Loss or dropped packets
    - Unordered packet delivery

- **TCP (Transmission Control Protocol)**
  - Built on top of IP to ensure reliable, ordered packet delivery.
  - Uses **handshakes** to establish and close connections.
  - **Message Components**
    - IP Header: From/To addresses
    - TCP Header: Ordering of packets, no. of packets etc
    - Data: Information to be shared

- **HTTP/HTTPS (Hypertext Transfer Protocol)**
  - Client requests, Server responds
  - Protocol for web communication using a **request-response model**.
  - Supports **verbs** like GET, POST, DELETE, PATCH/PUT, TRACE, OPTIONS, HEAD

---

## 2. Storage, Latency, and Throughput
- **Storage Types**
  - **Memory (RAM):** Fast but volatile(data lost on Power OFF), costly.
  - **Disk Storage:** Persistent but slower (used for databases). HDD & SSD.

- **Latency**
  - Affected by Client-Server distance.
  - Time taken for a client request to reach server (Correct).
  - Request-Response cycle = Latency (duration for client request to reach server) + Server's response generation time on received request.
  - Minimising Client-Server distance reduces latency. Efficient, optimum processing of request reduces server's response generation time (e.g., searching in a dict over an array).

- **Throughput**
  - Max capacity of a system. Amount of data processed in a given time (e.g., Mbps or requests/second).
  - Higher throughput implies better scalability improving performance.
  - System is limited by its lowest throughput (bottleneck) server/part.

---

## 3. System Availability
- **High Availability (HA)**
  - Achieved by reducing **single points of failure** by replication for redundancy.
  - Fault-tolerant system: robust to handle failures in the networks, databases, servers etc.

- **SLAs (Service Level Agreements)**
  - Commitments to uptime (e.g., 99.99% availability).
  - 99.9% uptime means 8.77 hours of DOWNTIME per year
  - 99.99% uptime means 52.6 mins DOWNTIME per year
  - 99.999% uptime means approx 5 mins DOWNTIME per year

- **Designing HA Systems**
  - Use **replication** and **load balancing** to ensure resilience.
  - Avoid Single Point of Failures, introduce redundancy.
  - Replication, Redundancy, Load Balancing, Horizontal Scaling, CAP, Cache, Databases, Sharding & Partitioning, Workers (parallelism utilising CPU cycles), Queues, Schedulers (for preprocessing), Client-Server distance (Latency)

---

## 4. Caching and CDNs
- **Caching**  
  - Stores frequently used data for fast access (reduces latency).
  - Reduces Database overload
  - Reduces Network calls to Database
  - Fast access compared to Disk/Database
  - Use cache miss and eviction strategies according to use-case

- **Content Delivery Networks (CDNs)**
  - Distribute content (preprocessed data) across regional/global servers to reduce latency and load on main server.
  - Acts as mini-servers to quickly serve requests from a closer geographical location.
  - Two types of CDNs: Push & Pull CDNs

---

## 5. Proxies
- **Forward Proxy**
  - Acts on behalf of a client (e.g., VPNs).
  - Hides client's IP, geographical location.

- **Reverse Proxy**
  - Works on behalf of a server, often handling **load balancing** and **security**.
  - Hides backend servers from public access, (internal servers) can have private IPs and no encryption etc for optimisation and free flow.

---

## 6. Load Balancing
- **Algorithms**  
  - Round-robin, Least connections, Load Based, Mixed Bag, Path or Service based, IP-hash (Geography), Weighted Round-robin, Server statuses, Alphabetic partition.
- **Use Case:** Distributes traffic across multiple servers to prevent overload.
- Mainly introduced to regulate n/w calls or communications like client-load balancer-backend, backend-load balancer-database, backend-load balancer-caches etc.
- Can act as rate limiter, reverse proxy server too.
- Ensure Load Balancers redundancy, fail-over, fail-back to achieve scalability and reduce fault-tolerance.
- Provides "Intelligent Request-Response routing".

## 7. Hashing
- **Hashing:** Puts universe of keys to fixed size bags.
- Collision Handling: Chaining, Open Addressing: Linear, Quadratic and Double Hashing/Probing.
- **Cons:**
  - server fails, traffic still gets routed to it.
  - High data redistribution with any change in node count, wastes previous caches.
  - No ring structure; applies hashes directly to data.

## 8. Consistent Hashing
- Ring structure; server is hashed and placed at multiple locations for uniform distribution/routing of requests.
- Servers at different locations in a ring can be called nodes
- A server/node fails, requests are redirected to the next node in loop.
- To achieve optimum uniformity in load distribution, hashed requests can be routed to hashed-server/node
- Introduced to backend-servers and caches.
- Addition or removal of servers has least impact on the system.

## 9. Databases
- **Relational**
  - Strictly enforced relationships/schema like rows and columns, datatypes, null, not-null, primary & foreign keys etc.
  - Schema: Classic relational database or formalized entity structure.
  - Data being inserted should conform to the predefined schema.
  - SQL: Structured Query Language, a language designed to interact with structured (relational) database.
  - Databases itself manages these queries(executes them) and returns matching results.
  - Follows ACID property
    - **Atomicity:** One operation fails, entire transaction fails. All or nothing. It must either be complete in its entirety or have no effect whatsoever.
    - **Consistency:** Every read operation receives the most recent write operation result. Full-synchronisation. It must conform to existing constraints in the database.
    - **Isolation:** you can "concurrently" (at the same time) run multiple transactions on a database, but the database will end up with a state that looks as though each operation had been run serially ( in a sequence, like a queue of operations). It must not affect other transactions
    - **Durability:** data stored in DB is persistent (ROM or disk), not in-memory (RAM).
- **Non-relational**
  - Less rigid, more flexible structure like key-value pairs.
  - Follows BASE
    - **Basically Available:** system guarantees availability.
    - **Soft State:** DB/table state may change over time, even without input.
    - **Eventual Consistency:** the system will become consistent over a period of time, given that the system doesn't receive input during that period.
  - At core, DB holds data in a hash-table-like structure (thus, extremely fast, simple and easy to use).
  - Perfect for use-cases like caching, environment variables, configuration files and session state etc.
  - Memcached (in-memory), DynamoDB (persistent storage), MongoDB (document DB)

---

- **Database Indexing**
  - **Definition**
    - A data structure added to databases to facilitate fast searches for specific attributes (fields).
    - Reduces lookup time compared to iterating through entire datasets.
  - **Example**
    - A database with 120 million records indexed on "age" allows fast queries for age-based filtering.
  - **Benefits**
    - Optimized lookup times.
    - Supported by both relational and non-relational databases.

---

- **Replication**
  - **Definition**
    - Duplication of databases for redundancy and high availability.
  - **Types of Replication**
    - **Synchronous Replication**
      - Data replicated simultaneously to replicas.
      - Ensures strong consistency.
      - Higher write response duration due to wait times.
    - **Asynchronous Replication**
      - Data replicated after confirming writes to the main database.
      - Better performance but eventual consistency.
  - **Considerations**
    - Synchronization intervals depend on use case.
    - Atomicity: Write to main DB fails if replica synchronization fails.
  - **Benefits**
    - High availability.
    - Redundancy to handle failures, prevent data loss.
    - Active usage of redundant system helps scale a system too.

---

- **Sharding**
  - **Definition**
    - Partitioning data into smaller, manageable chunks called "shards."
  - **Purpose**
    - Addresses scalability when replication alone cannot solve throughput or request-response cycle delays.
  - **Strategies**
    - **Row-based**: Each shard contains a fixed number of rows.
    - **Attribute-based**: Shards determined by specific attributes (e.g., region, customer ID).
  - **Benefits**
    - Improves performance by distributing data across **multiple smaller databases (shards)**.
  - **Use Cases**
    - Global applications with data partitioned by geographic location.

---

## 10. Leader Election
- **Definition**
  - Process of designating one server as the leader to perform specific tasks.
- **Purpose**
  - Avoid multiple servers performing conflicting actions (e.g., writes or API updates).
- **Key Features**
  - Detection of leader failure.
  - Re-election of a new leader in case of failure.
- **Challenges**
  - Maintaining synchronization of data, state, and operations among servers.
  - Handling partial network outages and disconnected servers.
- **Consensus Algorithms**
  - Used to agree on which server is the leader.
  - Example: Blockchain-inspired consensus for leader selection.
- **Tools**
  - **etcd**
    - Key-value store for high availability and strong consistency.
    - Stores the current leader as a key-value pair.
    - Relied upon for accurate "source of truth" in leader elections.

---

## Connections and Concepts
- **Indexing and Replication**
  - Indexing optimizes data access speed; replication ensures availability, fault tolerance, prevents data loss.
- **Replication and Sharding**
  - Replication duplicates data for redundancy while, sharding stores partitioned data into shards improving scalability.
- **Leader Election and Replication**
  - Leader election often operates on systems using replication to ensure one source controls updates.
- **Sharding and Indexing**
  - Indexing improves query performance within individual shards.

---

## Summary
- **Indexing**: Efficient querying.
- **Replication**: High availability.
- **Sharding**: Scalability.
- **Leader Election**: Coordination in redundant systems.

---

## Section 11: Polling, Streaming, Sockets

### Polling
- **Definition**
  - The client repeatedly requests data from the server at regular intervals.
- **Characteristics**
  - Simple and predictable.
  - Causes frequent server requests.
- **Downsides**
  - High network, server load, repeated requests add to throughput and rate-limiting.
  - Not truly real-time.
- **Use Case**
  - Driver location updates in a ride-hailing app every few seconds.
  - Payment confirmation update via Payment Gateways.

### Streaming
- **Definition**
  - Uses long-lived connections for real-time two-way data flow (e.g., WebSockets).
- **Key Features**
  - Server pushes updates as they occur.
  - Efficient for continuous updates.
- **Advantages**
  - Real-time data transmission.
  - Reduces overhead compared to polling.
- **Use Case**
  - Collaborative coding, online multiplayer games.
- **Comparison**
  - Polling pulls data; streaming pushes data.

---

## Section 12: Endpoint Protection

### Rate Limiting
- **Definition**
  - Limits the number of operations a client can perform in a time window.
  - Load Balancer: Fixed size / Leaky Bucket
- **Benefits**
  - Protects servers from abuse (e.g., DoS attacks).
  - Enforces usage tiers (e.g., free-tier API limits).
- **Key Considerations**
  - Fast limit checks with in-memory databases (e.g., Redis).
  - Distributed systems require consistent limit enforcement.

### Denial of Service (DoS) Protection
- **Problem**
  - Overwhelms servers with excessive requests.
- **Mitigation**
  - Rate limiting handles simple DoS attacks.
  - Sophisticated protection needed for Distributed DoS (DDoS).

---

## Section 13: Messaging & Pub-Sub

### Messaging Systems
- **Definition**
  - Facilitate communication between distributed system components.
- **Importance**
  - Ensures reliable, asynchronous data exchange.
- **Use Case**
  - Ticketing systems: notifying email services about confirmed bookings.

### Pub/Sub Model
- **Components**
  - **Publisher**: Sends messages.
  - **Subscriber**: Receives messages.
  - **Topics**: Channels for specific message types.
  - **Messages**: Data being communicated.
- **Advantages**
  - Decouples publishers and subscribers.
  - Enables event-driven architectures.
- **Guarantees**
  - "At least once" delivery ensures no message loss.

### Idempotency
- **Definition**
  - Operations produce the same result if performed multiple times.
- **Use Case**
  - Payment systems avoid duplicate charges.
- **Non-Idempotent Examples**
  - Social media comments or likes.

---

## Section 14: Smaller Essentials

### Logging
- **Purpose**
  - Tracks system events for debugging, monitoring, and analytics.
- **Key Features**
  - Time-series data format.
  - Used for audits and performance tuning.

### Monitoring
- **Purpose**
  - Converts log data into insights through dashboards and alerts.
- **Tools**
  - Specialized databases for time-series data.
  - Alerts for critical metrics like latency or error rates.

### Alerting
- **Definition**
  - Automated notifications for threshold breaches.
- **Examples**
  - High response times, error spikes.

---

## Connections Across Topics
- **Polling vs. Streaming**
  - Polling suits periodic updates; streaming is real-time.
- **Messaging & Logging**
  - Logs track system events; messaging queues tasks based on these events.
- **Endpoint Protection & Pub/Sub**
  - Rate-limiting protects publishers in event-driven systems.
- **Monitoring & Idempotency**
  - Alerts ensure consistent outcomes in idempotent operations.



---

For more details, explore the full article [here](https://www.freecodecamp.org/news/systems-design-for-interviews/).