# CS75 Harvard Lecture 9: Scalability — Complete Reference

> **Source:** CS75 (Summer 2012) Lecture 9 by David Malan — Harvard Web Development  
> **Supplemented by:** Designing Data-Intensive Applications (Kleppmann), System Design Interview (Alex Xu), Google SRE Book  
> **Purpose:** Complete self-contained reference — no need to re-watch the video.

---

## Table of Contents

1. [Web Hosting Landscape](#1-web-hosting-landscape)
2. [Vertical Scaling](#2-vertical-scaling)
3. [Storage Hardware & RAID](#3-storage-hardware--raid)
4. [Horizontal Scaling](#4-horizontal-scaling)
5. [Load Balancing — DNS Round Robin](#5-load-balancing--dns-round-robin)
6. [Load Balancing — Dedicated Load Balancers](#6-load-balancing--dedicated-load-balancers)
7. [Sticky Sessions & Session Management](#7-sticky-sessions--session-management)
8. [PHP Acceleration & Opcode Caching](#8-php-acceleration--opcode-caching)
9. [Caching Strategies](#9-caching-strategies)
10. [MySQL Storage Engines](#10-mysql-storage-engines)
11. [Database Replication](#11-database-replication)
12. [Database Partitioning](#12-database-partitioning)
13. [High Availability (HA)](#13-high-availability-ha)
14. [Multi-Tier Architecture Diagram](#14-multi-tier-architecture-diagram)
15. [Multi-Datacenter & Availability Zones](#15-multi-datacenter--availability-zones)
16. [Security & Firewalling](#16-security--firewalling)
17. [Supplementary: Topics Beyond the Lecture](#17-supplementary-topics-beyond-the-lecture)
18. [Key Design Principles](#18-key-design-principles)

---

## 1. Web Hosting Landscape

### Shared Hosting
- Multiple customers share the same physical server.
- Provider banks on the fact that ~90% of customers use very little resource.
- "Unlimited bandwidth/storage" claims are misleading — resources are contended.
- **Implication:** You cannot control OS configuration, cannot tune for scale.
- **Risk:** Noisy neighbour problem — one heavy user degrades everyone else.

### Virtual Private Server (VPS)
- A hypervisor (VMware, Citrix, KVM, Xen) slices one powerful physical server into multiple isolated VMs.
- Each customer gets their own OS image (Ubuntu, Fedora, etc.).
- **Privacy note:** The VPS provider still has physical access. They can reboot into single-user mode and access your files. True privacy requires owning your own hardware.
- **Examples:** Linode, DigitalOcean, AWS EC2, Rackspace Cloud.
- **Cost:** ~$50–$200+/month for dedicated slices vs ~$10/month shared hosting.

### Dedicated / Bare-Metal Server
- You rent or own an entire physical machine.
- Full control of hardware, OS, network.
- Used when VPS overhead is unacceptable (high I/O workloads, compliance needs).

### Cloud (AWS EC2 / Elastic Compute Cloud)
- Self-service VPS provisioning at cents-per-minute pricing.
- Spawn machines programmatically → scale out on traffic spike, terminate when idle.
- Supports auto-scaling groups, load balancers, managed DBs.
- **Key benefit:** No upfront CapEx; pay for what you use.

> **DDIA Note:** The distinction between shared, VPS and bare-metal maps to the infrastructure layer. At scale, most companies use IaaS (Infrastructure as a Service) like AWS/GCP/Azure and build their own application layer on top.

---

## 2. Vertical Scaling

**Definition:** Buy a bigger, faster single server — more CPU cores, more RAM, faster disk.

### When it makes sense
- Early stage, traffic is manageable on a single server.
- Simplest architecture — no distributed systems complexity.
- Stateful systems that are hard to shard (e.g., single-writer databases).

### CPU / Core context
- Modern servers have multiple CPUs, each with multiple cores.
- A quad-core machine can literally execute 4 threads simultaneously (true parallelism).
- Web servers (Apache, Nginx) spawn multiple worker processes/threads to handle many concurrent requests — true parallelism on multi-core.
- OS scheduler time-slices on single-core, creating the *illusion* of parallelism.

### Limitations
- **Hard ceiling:** Hardware has physical limits. You cannot buy a machine with infinite RAM or CPU.
- **Cost curve:** Performance improvements become non-linear with cost at the high end.
- **Single point of failure:** One big machine is still one machine. If it dies, everything dies.
- **Downtime for upgrades:** Adding RAM/CPU often requires a reboot.

> **Alex Xu (System Design Interview):** Vertical scaling is simple but has a hard upper limit. It is a good first step for startups but should be complemented with horizontal scaling as traffic grows.

---

## 3. Storage Hardware & RAID

### Hard Drive Technologies
| Type | Speed | Cost | Notes |
|------|-------|------|-------|
| Parallel ATA / IDE | 7200 RPM | Low | Obsolete, found only in old machines |
| SATA | 7200 RPM | Low-Medium | Standard for consumer desktops/laptops |
| SAS (Serial Attached SCSI) | 10,000–15,000 RPM | High | Used in servers; 2x faster than SATA |
| SSD | No moving parts | High | Fastest; no rotational latency; smaller max capacity at time of lecture |

- **Implication:** Databases are I/O intensive (every write and many reads touch disk). Upgrading to SAS or SSD directly improves DB throughput.
- Malan's advice: Put your database on the fastest disks your budget allows; use cheaper disks for static files.

### RAID (Redundant Array of Independent Disks)

RAID uses multiple physical drives to achieve either **performance** or **redundancy** (or both).

#### RAID 0 — Striping (Performance, No Redundancy)
- Data is split ("striped") across 2+ drives.
- Writes alternate between drives: chunk A → drive 1, chunk B → drive 2, etc.
- **Result:** Write speed roughly doubles with 2 drives.
- **Risk:** If either drive fails, ALL data is lost. Zero redundancy.
- **Use case:** Scratch/temporary data where speed matters more than durability.

#### RAID 1 — Mirroring (Full Redundancy, No Extra Speed)
- Every write is duplicated to both drives simultaneously.
- Either drive can fail; data is still intact on the other.
- **Recovery:** Buy a new drive, plug it in → array auto-rebuilds (sometimes without downtime, hot-swap).
- **Cost:** You sacrifice 50% of raw capacity for redundancy.
- **Use case:** OS drive, critical single-server data.

#### RAID 5 — Striping + Distributed Parity (Balance)
- Requires ≥ 3 drives.
- Data is striped across all drives; one drive's worth of space is used for parity (spread across all drives, not on a dedicated drive).
- **Tolerance:** Any 1 drive can fail without data loss.
- **Usable capacity:** (N-1)/N of total raw capacity. 5×1TB drives → 4TB usable.
- **Recovery:** Replace failed drive; array rebuilds from parity.
- **Use case:** General-purpose server storage.

#### RAID 6 — Striping + Double Parity
- Requires ≥ 4 drives.
- Two drives' worth of parity, spread across all drives.
- **Tolerance:** Any 2 drives can fail simultaneously without data loss.
- **Cost:** 2 drives of capacity sacrificed.
- **Use case:** Large arrays where simultaneous drive failure probability is non-negligible (disks in same batch may fail together).

#### RAID 10 (1+0) — Striping + Mirroring
- Requires ≥ 4 drives (even number).
- Data is striped across mirrored pairs.
- **Tolerance:** Any 1 drive per mirror-pair can fail (potentially multiple drives if in different pairs).
- **Performance:** Read and write speeds benefit from both striping and mirroring.
- **Cost:** 50% capacity overhead (same as RAID 1).
- **Use case:** High-performance + high-availability (e.g., busy MySQL primary server).

#### Server-Grade Extras
- Enterprise servers have redundant power supplies — if one PSU fails, the other takes over while you hot-swap the dead one.
- Multiple RAM banks — ECC (Error-Correcting Code) RAM detects/corrects bit errors.
- Multiple NICs — bonded for throughput or failover.

> **DDIA Note:** RAID protects against disk hardware failure but is NOT a substitute for backups. RAID does not protect against accidental deletion, software bugs, or fire/flood. Always maintain off-site backups separately.

---

## 4. Horizontal Scaling

**Definition:** Instead of one big server, use many smaller/cheaper servers in parallel.

### Why it wins long-term
- Commodity hardware is cheaper per unit of compute than high-end servers.
- Failure of one node does not take down the whole system.
- You can add nodes incrementally as traffic grows.
- Geographic distribution becomes possible.

### The core challenge
When you have N backend web servers, incoming HTTP requests need to be routed to one of them. The client only knows a single hostname (e.g., `www.example.com`). Someone must decide which server handles each request.

**Solution: a Load Balancer** (see section 6).

---

## 5. Load Balancing — DNS Round Robin

### How it works
- In your DNS zone file (e.g., BIND), define multiple A records for the same hostname pointing to different server IPs.
- DNS server rotates through the list of IPs, returning a different one each time it is queried.

```
; BIND zone file snippet
www   IN  A  1.2.3.4
www   IN  A  1.2.3.5
www   IN  A  1.2.3.6
```

- When user queries DNS for `www.example.com`, they get one of the IPs. The next user gets the next IP. Eventually wraps around.

### DNS Round Robin — Demonstrated Live (Lecture)
- Malan ran `nslookup google.com` live — Google returns multiple IPs per DNS query.
- Google's actual load balancing is more sophisticated, but DNS-level round-robin is part of it.

### Advantages
- Zero extra hardware or software infrastructure needed.
- Trivial to configure.
- Works across geographies (different IPs can be in different data centers).

### Disadvantages

#### 1. Load imbalance
- Round-robin distributes by *count* of requests, not by *computational cost*.
- One server might get all the heavy users; another gets all lightweight requests.
- No mechanism to correct this imbalance.

#### 2. DNS caching breaks uniformity
- Browsers, OS resolvers, and upstream DNS caches cache DNS responses per their TTL (Time-To-Live).
- A user who gets back IP `1.2.3.4` will continue sending all their requests to that IP until the TTL expires (could be minutes, hours, or a day).
- This concentrates load on whichever server that user was assigned to — defeating the distribution goal.
- **TTL tuning:** You can set a short TTL (e.g., 60 seconds) to force more frequent DNS lookups, but this adds DNS query overhead and still doesn't give perfect instantaneous balance.

#### 3. No health awareness
- If server `1.2.3.5` crashes, DNS still hands out its IP.
- Users whose DNS resolves to the dead server get errors until TTL expires.

#### 4. Sessions break
- See Section 7 for full detail. Round-robin may send the same user's consecutive requests to different servers, breaking server-side session storage.

---

## 6. Load Balancing — Dedicated Load Balancers

### Architecture
```
Internet
    |
[Load Balancer]  ← single public IP
   / |   /  |  [WS1][WS2][WS3]  ← private IPs (10.x.x.x or 192.168.x.x)
```

- DNS returns the IP of the load balancer.
- Load balancer forwards each request to a chosen backend server.
- Backend servers have only private IP addresses — not directly reachable from the internet.
  - Security benefit: no external attacker can directly address a backend server.
  - IPv4 conservation benefit: you only consume one public IP.

### Load Balancing Algorithms

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| Round Robin | Cycle through servers in order | Simple, uniform requests |
| Weighted Round Robin | Assign more requests to more powerful servers | Heterogeneous hardware |
| Least Connections | Send to whichever server has fewest active connections | Long-lived connections (WebSockets) |
| Least Response Time | Send to server with lowest latency + fewest connections | Latency-sensitive |
| IP Hash | Hash client IP → always same server | Basic sticky sessions |
| Resource-based | Poll servers for CPU/memory; route to least loaded | Heavy, variable workloads |

### SSL Termination at the Load Balancer
- Handling SSL (HTTPS, port 443) requires expensive asymmetric cryptography.
- Offloading SSL to the load balancer means:
  - Only one SSL certificate needed (on the LB, not every backend).
  - Backend servers receive plain HTTP (port 80) — cheaper CPUs.
  - If your internal network is trusted (you control it), this is acceptable.
- **Risk:** Traffic inside the data center is unencrypted. If the LB is compromised, plaintext is exposed.
- **Alternative:** End-to-end encryption (TLS everywhere), used when compliance demands it (e.g., PCI-DSS).

### Software Load Balancers
- **HAProxy** (High Availability Proxy): Free, open-source. Supports TCP and HTTP. Used by GitHub, Reddit, Stack Overflow historically. Supports round robin, least connections, source IP hash, and more.
- **Nginx**: Also acts as a reverse proxy and load balancer. Very performant for HTTP.
- **AWS Elastic Load Balancer (ELB) / ALB / NLB**: Managed, auto-scaling, integrates with AWS ecosystem.
- **Linux Virtual Server (LVS)**: Kernel-level load balancing, very high performance.

### Hardware Load Balancers
- Vendors: Cisco, Citrix NetScaler, F5 BIG-IP, Barracuda.
- Expensive: A mid-range unit may cost $20,000–$100,000+. Enterprise pairs with support contracts can exceed $100k.
- Offload SSL at hardware speed. Advanced L7 routing. High availability built in.
- **Malan's take:** Often atrociously overpriced for what they do. Software solutions like HAProxy can achieve the same at zero cost for many use cases.

### Load Balancer Redundancy (Active-Active vs Active-Passive)
The load balancer itself is a single point of failure. You need two.

#### Active-Active
- Both load balancers are live simultaneously.
- Both accept traffic; DNS or another mechanism distributes across both.
- If one dies, the other continues to handle 100% of traffic.
- Heartbeat packets sent between the two at regular intervals (e.g., every second).
- If heartbeat stops: surviving LB detects failure and takes full ownership.

#### Active-Passive
- One LB is "active" (handling traffic), the other is "passive" (standby, monitoring).
- Passive constantly sends/checks heartbeats to active.
- If active goes silent → passive promotes itself:
  - Takes over the active LB's IP address (IP takeover / floating IP / VRRP).
  - Begins accepting traffic.
- **Brief downtime** during failover (seconds, not minutes).

---

## 7. Sticky Sessions & Session Management

### The Problem
In PHP (and most server-side frameworks), sessions are stored as files on the local filesystem (`/tmp/` directory by default). When the load balancer routes a user to a different server, that server has no knowledge of the user's session → the user appears logged out, their shopping cart is empty, etc.

### Solution 1: Shared Session Storage (Centralized State)

Extract sessions from individual web servers into a shared data store that all web servers can read/write.

**Options:**
- **MySQL database:** Store session data as rows; all web servers connect to the same DB. Simple if you already have MySQL; adds DB load for every request.
- **Dedicated file server (NFS):** Export a network filesystem; web servers write session files there. Adds network latency; single point of failure unless replicated.
- **Fiber Channel (FC):** Very fast, very expensive SAN (Storage Area Network) that presents shared block storage to multiple servers. Data-center grade.
- **Memcached:** Store session objects in RAM on a dedicated caching server accessible by all web servers. Fast, but volatile (sessions lost on restart unless persisted elsewhere).
- **Redis:** Like Memcached but with persistence options and richer data structures. Modern preferred solution.

**Problem introduced:** The shared storage server becomes a new single point of failure. Fix with RAID on that server (mitigates disk failure) + redundancy (replicated storage servers, master-master).

### Solution 2: Cookie-Based Sticky Sessions (LB-managed)

The load balancer inserts a special cookie into the HTTP response.

**Flow:**
1. First request arrives at LB. LB assigns user to backend server 1.
2. LB inserts a `Set-Cookie` header in the response with a large random token (e.g., `SERVERID=abc123xyzrandom`).
3. LB maintains a mapping table: `abc123xyzrandom → server_1`.
4. All subsequent requests from that user include the cookie → LB reads it → routes to server 1.
5. If server 1 dies: LB assigns the user to a new server; old cookie value maps to dead server. LB can detect failure and re-assign (brief disruption).

**Why not store the server IP directly in the cookie?**
- Exposes your internal network topology.
- IP addresses can change.
- Users could spoof the cookie to target specific backend servers.

**Downside:** If the user disables cookies, this mechanism breaks. Also, if the LB itself dies and the replacement doesn't have the same mapping table, users lose their server affinity.

### Solution 3: IP-based Hashing
- Hash the client's IP address → deterministically route to the same server.
- Simple, no state needed on the LB.
- **Problem:** NAT — many users behind the same NAT gateway have the same source IP → all routed to the same server (defeats load balancing).

### Modern Best Practice
- Store session state in a **shared, fast, redundant store** (Redis Cluster or DynamoDB) — decoupled from web servers entirely.
- Or move to **stateless authentication** (JWT tokens): the session state is encoded in a signed token the client carries. No server-side session storage needed at all.

> **DDIA (Chapter 6):** The cleanest scalable architecture makes servers stateless — all persistent state lives in the database tier. Stateless web servers can be added/removed at will.

---

## 8. PHP Acceleration & Opcode Caching

### The Problem
Interpreted languages (PHP, Python, Ruby) go through an extra step relative to compiled languages (C, C++):

1. Source file is read from disk.
2. Interpreter **parses** the source into an AST.
3. AST is **compiled** into internal **opcodes** (bytecode).
4. Opcodes are **executed** by the VM.

In vanilla PHP (without acceleration), steps 2–3 happen **on every single request** for every `.php` file. The compiled opcodes are thrown away afterward.

### Solution: Opcode Cache
An opcode cache intercepts step 2–3 and stores the resulting opcodes in shared memory. On subsequent requests, PHP skips parsing/compilation entirely and jumps straight to execution.

**Effect:** 2–5x or greater reduction in CPU time per request on typical PHP workloads.

**Caveat:** If you modify a `.php` source file, the stale opcodes must be invalidated. Opcode cache software detects file modification time changes automatically, or you can manually flush the cache after deployment.

**PHP Opcode Cache options mentioned:**
- APC (Alternative PHP Cache) — historically common
- eAccelerator
- XCache
- Zend Optimizer

**Modern (post-lecture) standard:**  
- **OPcache** — bundled into PHP since PHP 5.5. Enabled by default in most PHP distributions. No external install needed.

**Python parallel:**  
- `.pyc` files are Python's compiled bytecode, stored next to `.py` source files.
- Python automatically recompiles if the source file is newer than the `.pyc`.

> **Alex Xu:** Application-level performance tuning (opcode caching, query optimization, connection pooling) should come before adding hardware. Always profile first.

---

## 9. Caching Strategies

### Hierarchy of Speed (Slowest to Fastest Read)
```
Disk (spinning HDD) → SSD → Database (with indexes) → Memcached/Redis (RAM) → CPU Cache (L1/L2/L3)
```

### Strategy 1: Static File Caching (Craigslist Approach)
- Generate HTML once (when content is created/updated), write it as a static `.html` file on disk.
- Subsequent requests are served directly by the web server (Apache/Nginx) as static files — no PHP/Python execution, no DB query.
- Web servers are extremely fast at serving static bytes.

**Advantages:**
- Minimal server-side compute per page view.
- Works well for content that changes rarely (classifieds, articles).
- Craigslist still uses this model; reportedly handles enormous traffic with relatively little hardware.

**Disadvantages:**
- Disk space grows proportionally with number of pages.
- Redundancy: every page has the same header/footer HTML baked in (no templates at serve time).
- Redesigning the site requires regenerating potentially millions of files.
- Not suitable for truly dynamic content (user-specific pages, real-time data).

### Strategy 2: MySQL Query Cache
- MySQL can be configured to cache the results of SELECT queries in memory.
- Enable in `my.cnf`:
  ```ini
  [mysqld]
  query_cache_type = 1
  query_cache_size = 128M
  ```
- **How it works:** MySQL hashes the SQL string. On first execution, the result set is stored in the query cache. On identical subsequent queries (exact string match), the cached result is returned without hitting disk/indexes.
- **Invalidation:** If any row in any of the query's referenced tables changes (INSERT/UPDATE/DELETE), all cached queries touching that table are automatically invalidated.
- **Limitation:** Cache is per-server. High write rates invalidate the cache constantly, making it ineffective. Query cache was eventually **removed in MySQL 8.0** due to poor scalability under write-heavy workloads.

### Strategy 3: Memcached (Application-Level Cache)
Memcached is a free, open-source, in-memory key-value store. It runs as a separate server process and is accessible over TCP.

**Interface (PHP example):**
```php
$mc = new Memcache;
$mc->connect('localhost', 11211);

$user = $mc->get($user_id);  // Try cache first

if ($user === false) {
    // Cache miss: query the database
    $pdo = new PDO('mysql:host=db;dbname=app', $user, $pass);
    $stmt = $pdo->query("SELECT * FROM users WHERE id = ?");
    $stmt->execute([$user_id]);
    $user = $stmt->fetch(PDO::FETCH_ASSOC);

    // Populate the cache for next time (TTL = 3600 seconds)
    $mc->set($user_id, $user, 0, 3600);
}

// Use $user
```

**Data flow:**
1. Application checks Memcached for `key = user_id`.
2. **Cache hit:** Return cached value immediately. No DB query.
3. **Cache miss:** Query database. Store result in Memcached with a TTL. Return result.

**Eviction when RAM is full:**
- Memcached uses an **LRU (Least Recently Used)** eviction policy.
- When the cache is full and a new item must be inserted, the item that was accessed least recently is evicted.
- Accessing (getting) an item updates its "recently used" timestamp.
- **Effect:** Hot (frequently accessed) objects stay in cache; cold (rarely accessed) objects age out.

**Why Facebook used Memcached heavily:**
- Facebook is read-heavy: users browse many profiles and newsfeeds, but update their own profile infrequently.
- A profile that doesn't change can be served from Memcached cache for many reads without hitting MySQL.
- Facebook at peak operated thousands of Memcached servers.

**Memcached vs Redis:**

| Feature | Memcached | Redis |
|---------|-----------|-------|
| Data types | String/binary blobs only | Strings, Lists, Sets, Sorted Sets, Hashes, Streams |
| Persistence | No (RAM only) | Optional (RDB snapshots, AOF log) |
| Replication | No (client-side sharding only) | Yes (master-replica) |
| Clustering | Client-side sharding | Redis Cluster (native sharding) |
| Pub/Sub | No | Yes |
| Lua scripting | No | Yes |
| Best for | Pure caching | Caching + sessions + queues + leaderboards |

> **DDIA (Chapter 5):** Caching is a read-heavy optimisation. The key challenge is **cache invalidation** — knowing when to expire or update cached data. "Cache invalidation is one of the two hard problems in computer science" (the other being naming things and off-by-one errors).

---

## 10. MySQL Storage Engines

MySQL supports pluggable storage engines — the layer responsible for how data is physically stored on disk.

### InnoDB (Default)
- **Transactions:** Full ACID support (Atomicity, Consistency, Isolation, Durability).
- **Row-level locking:** Multiple writers can lock individual rows, not the whole table.
- **Foreign Keys:** Enforced at the DB level.
- **Crash recovery:** Uses redo logs (WAL — Write-Ahead Log).
- **Use case:** General-purpose OLTP. The right choice for most applications.

### MyISAM
- **No transactions.**
- **Table-level locking:** Any write locks the entire table. Concurrent writes queue up.
- **Faster reads** than InnoDB in some read-only workloads (due to simpler structure).
- **No foreign key support.**
- **Historically used** when transaction support wasn't needed and maximum read throughput was priority.
- **Modern recommendation:** Prefer InnoDB — row-level locking alone makes it better under most real-world write loads.

### MEMORY / HEAP Engine
- Data stored entirely in RAM.
- Extremely fast reads and writes.
- **All data is lost** on server restart or crash.
- **Use case:** Temporary tables, session caches, lookup tables that can be rebuilt from disk data.
- Alternative to Memcached for simple key-value caching within MySQL.

### ARCHIVE Engine
- Data is compressed on disk using zlib.
- Supports INSERT and SELECT but **no UPDATE or DELETE**.
- Much slower to query than InnoDB, but much smaller on disk.
- **Use case:** Log/audit tables where you write a lot, read rarely, and want to minimise disk usage. Web access logs, click-stream data.

### NDB (Network Database / MySQL Cluster)
- Designed for distributed, multi-node MySQL Cluster deployments.
- Data is partitioned and replicated across multiple data nodes in memory.
- Supports synchronous replication between nodes.
- **Use case:** Extremely high availability, telecom-grade requirements.
- Complex to operate; rarely used in web applications.

---

## 11. Database Replication

### Master-Slave Replication

```
         WRITES
         ──────▶
Client ──▶ [Master DB]
                │ Replication stream
         ┌──────┴──────┐
         ▼             ▼
    [Slave DB 1]  [Slave DB 2]
         ◀── READS ──▶
```

**How it works:**
1. All writes (INSERT/UPDATE/DELETE) go to the **master**.
2. Master writes changes to its **binary log** (binlog).
3. Each slave runs an I/O thread that reads from the master's binlog.
4. Each slave applies the changes to its own local storage via a SQL thread.
5. Result: slaves are eventually identical copies of the master.

**Benefits:**
- **Read scaling:** Route SELECT queries to any slave. Distribute read load across N slaves.
- **Redundancy:** If master dies, promote a slave to master.
- **Backups without downtime:** Take a cold backup from a slave without impacting the master.

**Tradeoffs:**
- **Replication lag:** Slaves apply changes asynchronously. A write to master may not yet be visible on a slave. For read-after-write consistency, route the writer's own subsequent reads to the master.
- **Single write path:** Master is still a single point of failure for writes. Failover requires either manual intervention or an automated orchestration tool (MHA, Orchestrator, ProxySQL).
- **Facebook's early use:** Directed all SELECTs to slaves, INSERTs/UPDATEs/DELETEs to master. With millions of users doing far more reading than writing, slaves absorbed the read load effectively.

### Master-Master Replication

```
Client A ──▶ [Master DB 1] ◀──▶ [Master DB 2] ◀── Client B
```

**How it works:**
- Both DB1 and DB2 accept writes.
- Each replicates its changes to the other bidirectionally.
- Either can go down; the other continues to serve both reads and writes.

**Benefits:**
- Eliminates the master as a single point of failure for writes.
- Geographic distribution: clients in different regions write to their nearest master; changes propagate globally.

**Challenges:**
- **Write-write conflicts:** If two clients write to the same row on different masters simultaneously, the system must detect and resolve the conflict. MySQL's native master-master doesn't auto-resolve — you must avoid concurrent writes to the same row, or use application-level conflict resolution.
- **Auto-increment collisions:** Two masters both inserting rows with auto-increment IDs will generate the same IDs. Fix by configuring masters to use different auto-increment offsets:
  - Master 1: `auto_increment_offset=1, auto_increment_increment=2` → IDs: 1, 3, 5, 7...
  - Master 2: `auto_increment_offset=2, auto_increment_increment=2` → IDs: 2, 4, 6, 8...

**Failover in code (PHP example):**
```php
try {
    $pdo = new PDO('mysql:host=db1;dbname=app', $user, $pass);
} catch (PDOException $e) {
    // db1 is down, fail over to db2
    $pdo = new PDO('mysql:host=db2;dbname=app', $user, $pass);
}
```

A cleaner solution is to put a load balancer or ProxySQL in front of the DB tier, so application code always talks to a single endpoint.

### Master-Slave + Master-Master Combined

```
[Master DB 1] ◀──▶ [Master DB 2]
      │                   │
 [Slave 1A]          [Slave 2A]
 [Slave 1B]          [Slave 2B]
```

- Both masters accept writes and replicate to each other.
- Each master has its own slaves for read scaling.
- Maximum redundancy + maximum read throughput.
- Complex to operate and tune.

> **DDIA (Chapter 5 — Replication):** Master-slave is called "single-leader replication." Master-master is "multi-leader replication." Both have well-studied tradeoffs. Leaderless replication (Dynamo-style) is a third paradigm, used in Cassandra and DynamoDB.

---

## 12. Database Partitioning

**Definition:** Split a dataset across multiple database servers based on some partition key.

### Why partition?
- A single master/slave set can only absorb so much write load (master is still a bottleneck).
- A single database server has a ceiling on storage.
- Partitioning splits both the write load and storage across N servers.

### Facebook's Early Partitioning Strategy
- `harvard.thefacebook.com` ran on one server.
- `mit.thefacebook.com` ran on a separate server.
- All Harvard data on server A; all MIT data on server B.
- Simple and effective early on.
- **Problem when crossing schools:** Poking someone at MIT from your Harvard account required cross-partition queries, which is expensive and complex.

### Partitioning by Last Name / Alphabetical Range
```
Partition 1 (A–M): users with last_name LIKE 'A%' ... 'M%'
Partition 2 (N–Z): users with last_name LIKE 'N%' ... 'Z%'
```
- Incoming requests: application or LB reads the user's last name from session/cookie → routes to the correct shard.
- Works well if the distribution is uniform.
- **Hotspot risk:** Uneven distribution (many users with names starting 'S') can cause imbalanced load.

### Partitioning by Hash
- Apply a hash function to the partition key (e.g., `user_id % N`).
- More uniform distribution than alphabetical.
- **Problem with naive hashing:** Adding or removing a server requires rehashing almost all keys (rebalancing is expensive). Solution: **Consistent Hashing** (see your `HashingVsConsistentHashing.md`).

### Vertical vs Horizontal Partitioning

| Type | Description | Example |
|------|-------------|---------|
| Vertical | Split by columns — different tables go to different servers | Users table on DB1, Orders table on DB2 |
| Horizontal | Split by rows — same table's rows distributed across servers | Users with ID 1–1M on DB1, 1M–2M on DB2 |

### Cross-Shard Queries
The main difficulty with horizontal sharding. A query like `SELECT * FROM orders JOIN users ON users.id = orders.user_id` becomes non-trivial if `orders` is on one shard and `users` is on another. Solutions:
- Denormalize: duplicate relevant user fields into the orders table.
- Application-level joins: query each shard separately, join in application code.
- Use a distributed SQL database (e.g., CockroachDB, Spanner, TiDB) that handles this transparently.

> **Alex Xu (Chapter 5):** Sharding is powerful but complex. Avoid it as long as you can. Before sharding, exhaust: caching, read replicas, vertical scaling, query optimization.

---

## 13. High Availability (HA)

**Definition:** A system or component that is continuously operational for a desirably long length of time.

### Key Concept: Heartbeats
- HA pairs of servers (load balancers, databases) send periodic **heartbeat packets** to each other.
- Typically every 1 second.
- If a server stops receiving heartbeats from its peer → assumes peer has failed → takes over.

### Load Balancer HA
- **Active-Active:** Both LBs are live simultaneously, splitting traffic. If one fails, the other absorbs 100%.
- **Active-Passive:** One LB is active. Passive monitors with heartbeats. On failure: passive does **IP takeover** — claims the VIP (Virtual IP) of the failed node. Traffic continues to flow to the same IP, now answered by the surviving node. Implemented using VRRP (Virtual Router Redundancy Protocol) or Keepalived.

### Database HA
- **Semi-synchronous replication (MySQL):** At least 1 slave must acknowledge receipt of a write before the master returns success to the client. Reduces risk of data loss on master failure.
- **Automatic failover tools:** Orchestrator (GitHub), MHA (Master High Availability Manager), ProxySQL — monitor the master, detect failure, promote a slave, reroute application traffic.
- **Group Replication / Galera Cluster:** Synchronous multi-master replication. Every node has a copy of all data. Quorum-based writes. Any node can accept writes and reads.

### Measuring Availability

| Availability | Downtime/year |
|-------------|---------------|
| 99% ("two nines") | ~3.65 days |
| 99.9% ("three nines") | ~8.7 hours |
| 99.99% ("four nines") | ~52 minutes |
| 99.999% ("five nines") | ~5.3 minutes |

> **Google SRE Book:** "Hope is not a strategy." HA requires deliberate design, runbooks, regular failover drills, and chaos engineering (intentionally killing components to test recovery).

---

## 14. Multi-Tier Architecture Diagram

The following describes the complete topology discussed near the end of the lecture:

```
                 INTERNET
                    │
           ┌────────┴────────┐
        [LB 1]           [LB 2]        ← Active-Active pair (heartbeat ↔)
           │                 │
   ┌───────┴──────┬──────────┘
   │              │
[WS 1]         [WS 2]                  ← Web servers (PHP/Python); private IPs
   │              │
   └──────┬───────┘
          │
    [DB Load Balancer]                 ← Routes MySQL traffic
          │
   ┌──────┴──────┐
[MySQL Master 1] ◀──▶ [MySQL Master 2] ← Master-Master replication
   │                        │
[Slave 1A]              [Slave 2A]     ← Read replicas
[Slave 1B]              [Slave 2B]
```

**Network layer:**
- All servers connected to two physical switches (redundant uplinks).
- Spanning Tree Protocol or equivalent prevents switching loops.

**Traffic flows:**
- HTTP/HTTPS (ports 80, 443): Internet → LB
- HTTP (port 80): LB → Web Servers (SSL terminated at LB)
- MySQL (port 3306): Web Servers → DB Load Balancer → MySQL Masters
- Replication stream: Master → Slaves (MySQL binlog, TCP 3306 internally)

---

## 15. Multi-Datacenter & Availability Zones

### The Building Itself is a Single Point of Failure
- Power outage, network disconnect, natural disaster, fire can take down the entire data center.
- RAID, HA pairs, redundant switches — all useless if power to the building is cut.

### Availability Zones (AWS Model)
- Amazon names them: `us-east-1a`, `us-east-1b`, `us-east-1c`, etc.
- Each AZ is a physically separate building (data center) within a geographic region.
- Different power sources, different network paths.
- **Goal:** Failure of one AZ should not affect others.
- **Reality:** Amazon has experienced correlated outages across multiple AZs — often due to shared control plane software or shared networking infrastructure at the region level.

### Geographic Regions
- Beyond AZs within a region, AWS/GCP/Azure have entire separate regions: `us-east-1` (Virginia), `us-west-2` (Oregon), `eu-west-1` (Ireland), `ap-south-1` (Mumbai), etc.
- True disaster resilience requires deploying across at least 2 regions.
- **Challenge:** Cross-region replication adds latency; maintaining consistency across regions is fundamentally hard (CAP theorem).

### DNS-Based Global Load Balancing
- When you have two data centers (or two AWS regions), DNS is used to distribute traffic geographically.
- DNS server returns the IP of the nearest data center based on:
  - **Geographic routing:** Client's IP → geo lookup → return nearest DC's IP.
  - **Latency-based routing (AWS Route 53):** Return the IP of the region with lowest measured latency to the client.
  - **Health-check-based failover:** If DC1's health check fails, all DNS responses point to DC2.

```
User in India           User in USA
     │                       │
     ▼                       ▼
DNS returns IP of        DNS returns IP of
Mumbai datacenter        Virginia datacenter
```

- **TTL and failover:** If a data center goes down and DNS is updated, clients with cached DNS responses won't get the new IP until their TTL expires. Setting low TTLs (30–60 seconds) speeds up failover at the cost of more DNS queries.

> **Malan's note:** Even with multi-AZ, some outages affect multiple zones. Amazon's east coast region has suffered multiple multi-AZ events. High-traffic websites like Reddit, Foursquare, Netflix were impacted during Amazon's major 2011 outage.

---

## 16. Security & Firewalling

### Principle of Least Privilege
Only allow the minimum network access necessary. Every open port is a potential attack surface.

### Firewall Rules for a Typical LAMP Web Stack

| Traffic | Source | Destination | Port | Protocol | Allow? |
|---------|--------|-------------|------|----------|--------|
| HTTPS from users | Internet | Load Balancer | 443 | TCP | ✅ |
| HTTP from users | Internet | Load Balancer | 80 | TCP | ✅ (redirect to 443) |
| SSH for admin | Your IP only | Servers | 22 | TCP | ✅ (IP restricted) |
| HTTP (internal) | Load Balancer | Web Servers | 80 | TCP | ✅ |
| MySQL queries | Web Servers | DB servers | 3306 | TCP | ✅ |
| MySQL from Internet | Internet | DB servers | 3306 | TCP | ❌ Never! |
| SSH between servers | Web Servers | DB servers | 22 | TCP | ❌ (minimal footprint) |

### Why Block MySQL from the Internet?
- If port 3306 is exposed publicly, attackers can:
  - Brute-force MySQL credentials.
  - Exploit MySQL vulnerabilities directly.
  - Exfiltrate your entire database without going through the application layer.
- Even if you think your MySQL password is strong: never expose database ports publicly. Defense in depth.

### Defense in Depth — Layered Security
Even if an attacker compromises a web server (e.g., via a PHP vulnerability), containment prevents lateral movement:
- Web server → can only reach DB server on port 3306.
- Web server → cannot SSH to DB server.
- Web server → cannot reach other internal services.
- Even if attacker controls a web server, they are "trapped" in a limited blast radius.

### SSL / TLS
- All traffic from users to the load balancer should be HTTPS.
- SSL termination at the LB is common for performance.
- Internal traffic (LB → web servers) can be HTTP if the internal network is trusted.
- For sensitive data (medical, financial), consider end-to-end TLS.

### VPN for Admin Access
- Rather than exposing SSH port 22 globally, put the data center behind a VPN.
- Admins connect to the VPN first, then SSH internally.
- Reduces attack surface dramatically.

### Database Security Best Practices (Beyond Lecture)
- Use separate DB users per application with minimal permissions (no `GRANT ALL`).
- Never use the root MySQL user in application code.
- Enable MySQL SSL for client-server connections if required.
- Regularly rotate credentials.
- Audit logs for all queries touching sensitive tables.

---

## 17. Supplementary: Topics Beyond the Lecture

These topics are referenced in related files in your repo and are closely related to the lecture material.

### Content Delivery Networks (CDNs)
- Geographically distributed servers that cache **static assets** (images, CSS, JS, videos) at edge locations close to users.
- User requests for `cdn.example.com/image.jpg` are served from the nearest PoP (Point of Presence).
- **Reduces:** Latency, origin server load, bandwidth costs.
- **Providers:** Cloudflare, Fastly, AWS CloudFront, Akamai.
- **Origin pull vs Push:** CDN can pull assets from origin on first request (lazy), or you pre-push assets to the CDN.
- Works well with the static file caching approach from section 9 — push generated HTML to CDN.

### Asynchronous Processing & Message Queues
- For long-running tasks (email sending, image resizing, report generation): don't make the user wait.
- Accept the request, put a message on a queue (RabbitMQ, Kafka, SQS, Celery), return `202 Accepted` immediately.
- Background workers consume from the queue and process tasks.
- **Benefits:** Decoupling, retry on failure, load levelling, parallelism.
- **Challenge:** "Exactly-once delivery" — the Two Generals' Problem. Hard guarantees require distributed transactions or idempotent consumers.

### CAP Theorem
- A distributed system can guarantee at most 2 of:
  - **C**onsistency: Every read gets the most recent write.
  - **A**vailability: Every request gets a response (no timeout).
  - **P**artition Tolerance: System operates even if messages between nodes are lost.
- Network partitions are unavoidable in real distributed systems → always P.
- Real choice is **CP vs AP**:
  - CP (Consistency + Partition Tolerance): Zookeeper, HBase, MongoDB (majority write).
  - AP (Availability + Partition Tolerance): Cassandra, CouchDB, DynamoDB (eventual consistency).
  - Traditional RDBMS are typically CA (assume no partition — single-node or tightly coupled).

> **DDIA (Chapter 9):** CAP theorem is a simplification. In practice, consistency is a spectrum (linearizability → sequential consistency → causal consistency → eventual consistency), and the tradeoffs are nuanced.

### Consistent Hashing
- Standard modular hashing (`key % N`) breaks on node addition/removal: nearly all keys must be remapped.
- Consistent hashing places both nodes and keys on a conceptual ring (hash space 0 to 2^32).
- A key maps to the first node clockwise from its position on the ring.
- Adding/removing a node only remaps O(K/N) keys (K = keys, N = nodes).
- Virtual nodes (multiple positions per physical node) improve load distribution.
- Used by: Amazon Dynamo, Apache Cassandra, Memcached clients, Nginx upstream hashing.

### Performance Metrics
- **Throughput:** Requests per second (RPS) or transactions per second (TPS). Measures capacity.
- **Latency:** Time for one request to complete. Measure at percentiles:
  - P50 (median): typical experience.
  - P95: experience of the slowest 5%.
  - P99 ("tail latency"): slowest 1%. Critical for user-facing services.
  - Optimising for average latency can hide severe tail latency problems.
- **Saturation:** How close are resources to their limit? CPU at 90% → near saturation.
- **Error rate:** Percentage of failed requests.
- **Amdahl's Law:** The speedup from parallelism is limited by the serial fraction of the workload. If 5% of your code is serial, maximum speedup from infinite parallelism is 20x.

### Connection Pooling
- Opening a new TCP connection + MySQL authentication handshake per query is expensive.
- Connection poolers (PgBouncer for PostgreSQL, ProxySQL for MySQL) maintain a pool of pre-established connections.
- Applications borrow a connection from the pool, use it, return it.
- Reduces connection overhead dramatically at high concurrency.

---

## 18. Key Design Principles

From the lecture + supplementary resources:

1. **There is no free lunch.** Every architectural decision trades off something. Caching speeds reads but adds staleness risk. Replication adds availability but adds complexity and replication lag.

2. **Solve the current problem, plan for the next order of magnitude.** Don't over-engineer on day 1. A startup on shared hosting is fine. Premature sharding before you have the traffic is a waste.

3. **Design for failure.** Every component will eventually fail. Redundancy at every tier: load balancers, web servers, database, storage, networking, power, even buildings.

4. **Single Points of Failure are debt.** Any component that can bring down the entire system is a ticking clock. Enumerate all SPOFs and eliminate them one by one as you scale.

5. **Measure before you optimise.** "We should forget about small efficiencies, say about 97% of the time: premature optimisation is the root of all evil." (Knuth). Profile first. Add caching where data shows it helps.

6. **Stateless web tier.** Keep web servers stateless — all persistent state lives in the DB/cache tier. This makes horizontal scaling of web servers trivial.

7. **Principle of Least Privilege.** Only open the network ports and grant the permissions strictly needed. Contain blast radius.

8. **Scaling is iterative.** Your architecture at 1,000 users will look very different from your architecture at 10 million users. Build systems that can evolve, not systems that must be thrown away and rebuilt.

9. **Cache at every layer.** Browser cache → CDN → Application cache → DB query cache → Connection pool cache. Each layer catches a different class of redundant work.

10. **DNS TTL is a tool.** For geographic failover and rolling deploys, short TTLs give you faster propagation. For stable production IPs, longer TTLs reduce DNS query load. Balance deliberately.

---

## Quick Reference: Technologies Mentioned

| Technology | Category | Notes |
|------------|----------|-------|
| Apache / Nginx | Web Server | Apache multi-process; Nginx event-driven, better under concurrency |
| HAProxy | Software LB | Free, TCP + HTTP, highly configurable |
| AWS ELB / ALB | Managed LB | Application Load Balancer = L7; NLB = L4 |
| Memcached | Cache | In-memory KV store, LRU eviction, volatile |
| Redis | Cache + Store | Persistent options, rich data types |
| MySQL InnoDB | RDBMS | ACID, row-level locking, default engine |
| MySQL MyISAM | RDBMS | No transactions, table-level locks |
| MySQL MEMORY | RDBMS | RAM-only, volatile, fast |
| MySQL Archive | RDBMS | Compressed, write-heavy log storage |
| MySQL NDB | Clustered RDBMS | Distributed, in-memory, HA |
| RAID 0 | Storage | Striping, performance |
| RAID 1 | Storage | Mirroring, redundancy |
| RAID 5 | Storage | Striping + parity, 1-drive fault tolerance |
| RAID 6 | Storage | Striping + double parity, 2-drive fault tolerance |
| RAID 10 | Storage | Striping + mirroring, performance + redundancy |
| SAS drives | Storage | 15,000 RPM, server-grade |
| SSD | Storage | No moving parts, fastest |
| OPcache | PHP | Built-in opcode cache since PHP 5.5 |
| BIND | DNS Server | Popular authoritative DNS, supports multiple A records |
| NFS | Filesystem | Network File System, shared storage |
| VRRP / Keepalived | HA Protocol | Virtual IP failover for LB pairs |
| AWS EC2 | IaaS | Virtual machines, pay-per-minute |
| AWS Route 53 | DNS | Geo, latency, health-check-based routing |

---

*Last updated: March 2026 — synthesised from CS75 Lecture 9 transcript, DDIA (Kleppmann), System Design Interview (Alex Xu), Google SRE Handbook.*
