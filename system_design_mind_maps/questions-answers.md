# Quest-1. 
    Partitioning vs Sharding:
    As I understand:
    1. Partition: chunk of data, can be stored in single database (chunking into multiple tables) or shards
    2. Shards: chunked databases (with or without master-slave redundancy etc)

    Replication:
    1. It basically duplicates data (either in single database having multiple tables or master-slave architecture achieving redundancy)
    2. How does it achieve atomicity? - Write to master Database fails if replica synchronisation fails - is it applicable to both Synchronous and Asynchronous replication (both) or one of them? Share steps to achieve atomicity in Asynchronous replication.

    Database Indexing:
    1. Share internal working/algorithm that facilitates fast searches using Database Indexing. Example: a query on "age" column: select * from Customer where age<50
    2. Does Database Indexing work for non-relational databases too? If yes, list databases featuring this, algorithms/internal working or architecture to achieve this.

    Consistent Hashing:
    1. Servers are hashed and placed at multiple locations in ring structure. Share algorithm/s or hashing mechanism/s facilitating this. 
    2. What happens when a server is removed or added? Add an animated ring structure allowing me to remove or add a server and you sharing its internal working when this happens.

    Cache vs RAM: 
    1. share their similarities and differences, keep basic, intermediate, advanced level details.
    2. consider their hardware, placement in a scalable system and a computer, cost, speed etc.

    Add anything you think is related and I may've missed.
    ## Quick roadmap

    To keep this focused, let’s structure your questions into 5 chunks:

    1. Partitioning vs sharding  
    2. Replication and atomicity  
    3. Indexing internals (relational + NoSQL)  
    4. Consistent hashing and node changes  
    5. Cache vs RAM in the memory hierarchy  

    We’ll hit each at a systems-design level, not just DB-theory level.

    ***

    ## 1. Partitioning vs sharding

    Your mental model is almost there; just one refinement.

    - **Partitioning**: Split a table into logical pieces (partitions) but still within one database instance. Often used for manageability and query performance (e.g., by date, range, hash). [dev](https://dev.to/devcorner/sharding-vs-partitioning-vs-replication-a-complete-guide-17b2)
    - **Sharding**: Also a kind of horizontal partitioning, but partitions are spread across **multiple database instances / servers**. Each shard is an autonomous DB (its own logs, indexes, etc.). [hazelcast](https://hazelcast.com/foundations/distributed-computing/sharding/)

    So:

    - Every shard contains partitions of data, but “partitioning” doesn’t inherently imply “on different servers”; “sharding” does. [pingcap](https://www.pingcap.com/article/sharding-vs-partitioning-a-detailed-comparison/)
    - In practice:  
    - Partitioning = “split big table into chunks inside one DB”  
    - Sharding = “split dataset across many DBs/nodes, each node owns some partitions”

    Question for you: In a SaaS multi-tenant app, when would you prefer **sharding by customer_id** over just **partitioning by customer_id on one big DB**, and why?

    ***

    ## 2. Replication and atomicity

    ### What replication does

    - **Replication** = keeping multiple copies (primary + replicas) in sync, mainly for availability, durability, and read scaling, not for query speed on the primary itself. [aerospike](https://aerospike.com/blog/sync-vs-async-replication/)
    - Forms: master–slave (primary–replica), multi-master, synchronous vs asynchronous. [adamdjellouli](https://adamdjellouli.com/articles/databases_notes/09_database_replication/04_synchronous_vs_asynchronous_replication)

    ### Atomicity vs replication mode

    Atomicity is a **transaction property on a single node** (all-or-nothing). Replication interacts with this, but they’re distinct concerns.

    #### Synchronous replication

    - Write flow (simplified):  
    1. Client sends transaction to primary.  
    2. Primary writes to its log / memory.  
    3. Primary sends changes to replicas.  
    4. Replicas persist and acknowledge.  
    5. Only after all required replicas ACK, primary commits and returns success. [nakivo](https://www.nakivo.com/blog/synchronous-vs-asynchronous-replication-strategy/)

    - Effect:  
    - Transaction is **atomic across replicas** in the sense that “commit” only happens if local + replicas succeed. [aerospike](https://aerospike.com/blog/sync-vs-async-replication/)
    - If a replica cannot acknowledge, the write fails or blocks (depending on configuration). [nakivo](https://www.nakivo.com/blog/synchronous-vs-asynchronous-replication-strategy/)
    - This is how systems get strong consistency and ACID-like behavior even with replication. [aerospike](https://aerospike.com/blog/sync-vs-async-replication/)

    #### Asynchronous replication

    - Write flow:  
    1. Client sends transaction to primary.  
    2. Primary commits locally (log + data).  
    3. Primary returns success to client.  
    4. Later, replication sender ships changes to replicas, which apply them. [adamdjellouli](https://adamdjellouli.com/articles/databases_notes/09_database_replication/04_synchronous_vs_asynchronous_replication)

    - Consequences:  
    - Atomicity is guaranteed **locally on the primary**, not across replicas at commit time. [nakivo](https://www.nakivo.com/blog/synchronous-vs-asynchronous-replication-strategy/)
    - Replicas lag; they see a prefix of the committed history (eventual consistency). [aerospike](https://aerospike.com/blog/sync-vs-async-replication/)

    So: “Write to master fails if replica sync fails” is characteristic of **synchronous** replication. In asynchronous replication, you normally **do not** block or fail the client because a replica is behind; otherwise you lose the performance benefit. [adamdjellouli](https://adamdjellouli.com/articles/databases_notes/09_database_replication/04_synchronous_vs_asynchronous_replication)

    ### Achieving “good enough” atomicity in async

    You can’t truly have “atomic commit across primary + replicas” and still be fully async, but you can reduce risk:

    - **Durable logging on primary**: ensure WAL / commit log is fsynced before ACK. [aerospike](https://aerospike.com/blog/sync-vs-async-replication/)
    - **Replication from log**: replicas pull from durable WAL so no committed transaction is skipped; they are just delayed. [aerospike](https://aerospike.com/blog/sync-vs-async-replication/)
    - **Semi-sync / quorum variants**:  
    - Require at least 1 replica ACK before commit returns (MySQL semi-sync, some PostgreSQL quorum commit configs).  
    - This is between full sync and full async. [nakivo](https://www.nakivo.com/blog/synchronous-vs-asynchronous-replication-strategy/)
    - **Client read-routing rules**:  
    - “Read your own writes” from primary only, or from a replica with a known replication-lag bound. [aerospike](https://aerospike.com/blog/sync-vs-async-replication/)
    - **Failover rules**:  
    - Promote replica only if it is up to latest confirmed position (e.g., compare LSNs). [aerospike](https://aerospike.com/blog/sync-vs-async-replication/)

    Question for you: In a global system with high write traffic and latency between regions, what tradeoff would you make between **sync** and **async** replication, and why?

    ***

    ## 3. Database indexing internals

    ### How indexes speed up queries

    Most general-purpose DBs use **B-tree / B+ tree indexes** as the default. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)

    Key ideas (B+ tree):

    - Data is stored in **sorted order** by key (e.g., age). [tencentcloud](https://www.tencentcloud.com/techpedia/129146)
    - Tree is **balanced**: all leaves at same depth → O(log N) search. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    - Inner nodes hold key ranges and child pointers; leaves hold pointers (or actual row locations). [tencentcloud](https://www.tencentcloud.com/techpedia/129146)

    For `SELECT * FROM Customer WHERE age < 50` with an index on `age`:

    1. Traverse the B+ tree to find the first leaf where `age` becomes < 50 (depends on direction and layout). [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    2. From that leaf, scan contiguous leaf nodes until `age >= 50` or end of tree. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    3. For each entry, follow the row pointer (e.g., to heap table) to fetch full row. [geeksforgeeks](https://www.geeksforgeeks.org/dbms/indexing-in-databases-set-1/)

    Because leaves are linked (linked list of leaf pages), range scans like `< 50` or `BETWEEN 20 AND 40` are efficient. [geeksforgeeks](https://www.geeksforgeeks.org/dbms/indexing-in-databases-set-1/)

    Other index structures:

    - **Hash indexes**: use hash(key) → bucket; extremely fast for exact equality (`age = 30`), but poor for ranges. [tencentcloud](https://www.tencentcloud.com/techpedia/129146)
    - **Bitmap indexes**: compressed bitmaps per value; good for low-cardinality columns in analytic DBs. [geeksforgeeks](https://www.geeksforgeeks.org/dbms/indexing-in-databases-set-1/)

    ### Indexing in non-relational DBs

    Many NoSQL systems absolutely rely on indexes, often similar structures:

    - **MongoDB**: B-tree-like indexes (typically B-trees/B+ trees) for single-field, compound, and text indexes. [tencentcloud](https://www.tencentcloud.com/techpedia/129146)
    - **Cassandra**: SSTables with sorted keys; partition key index + Bloom filters, plus secondary indexes (also using B-trees or similar structures depending on implementation). [tencentcloud](https://www.tencentcloud.com/techpedia/129146)
    - **Elasticsearch / OpenSearch**: inverted index (term → postings list of documents); not B-trees but highly optimized for text search. [tencentcloud](https://www.tencentcloud.com/techpedia/129146)
    - **Redis**: uses in-memory structures (hash tables, skip lists for sorted sets), which act like indexes. [tencentcloud](https://www.tencentcloud.com/techpedia/129146)

    So indexing isn’t “relational-only”; the core idea—auxiliary data structure mapping searchable keys → locations—appears everywhere. [tencentcloud](https://www.tencentcloud.com/techpedia/129146)

    Question for you: Given `WHERE age < 50 AND city = 'London'`, what kind of index design (single composite index? separate indexes?) would you consider, and why?

    ***

    ## 4. Consistent hashing

    ### Basic algorithm

    In classic consistent hashing (as in Dynamo, Memcached etc.):

    1. Choose a hash function (e.g., `SHA-1`, `MD5`, or a fast 64-bit hash) that outputs a large integer space. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    2. Interpret that space as a **ring** (0 to 2^m − 1, wrap-around).  
    3. Hash each server’s ID (e.g., “serverA”) to one or more points on the ring → **virtual nodes**. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    4. Hash each key to a point on the same ring. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    5. A key belongs to the first server encountered clockwise from its hash position. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)

    Virtual nodes (multiple positions per physical server) smooth out load imbalance and make rebalancing more even. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)

    ### When a server is added

    Conceptually:

    1. New server S is hashed to multiple positions on the ring. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    2. For each of S’s positions, S becomes the first node clockwise for the keys in that ring segment. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    3. Only keys in those affected segments move from their previous owner to S; other keys remain untouched. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)

    In practice: key movement ≈ O(1/N) of the keys when adding 1 of N nodes.

    ### When a server is removed

    1. Remove all of that server’s positions from the ring. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    2. Keys that mapped to those positions now map to the next node clockwise. [pingcap](https://www.pingcap.com/article/understanding-b-tree-and-hash-indexing-in-databases/)
    3. Again, only keys previously owned by that server are remapped.

    An animated ring would show “intervals” being re-assigned from one node to another, but the core internal working is: only affected hash ranges change ownership.

    Question for you: Suppose you have 4 nodes and you add a 5th. Why is it important to use **virtual nodes** instead of just one hash position per node?

    ***

    ## 5. Cache vs RAM

    Think in terms of the **memory hierarchy**: small & fast near the CPU; large & slow farther away. [scribd](https://www.scribd.com/document/926702258/q21)

    ### Basic similarities

    - Both hold data for the CPU to read/write.  
    - Both are volatile (lose content when power is off). [scribd](https://www.scribd.com/document/926702258/q21)
    - Both are implemented as semiconductor memory and form layers of the memory hierarchy. [dev](https://dev.to/sachin_tolay_052a7e539e57/understanding-cpu-cache-organization-and-structure-4o1o)

    ### Key differences

    | Aspect        | CPU Cache                             | RAM (Main Memory)                      |
    | --------------| --------------------------------------| ---------------------------------------|
    | Typical tech  | SRAM                                  | DRAM                                   |
    | Location      | On-CPU or very close (L1/L2/L3)       | On DIMMs on motherboard                |
    | Purpose       | Hide latency of RAM, exploit locality | Main working memory for programs       |
    | Size          | KB–tens of MB                         | GB–TB                                  |
    | Latency       | Few CPU cycles (L1), tens (L2/L3)     | Hundreds of cycles                     | 
    | Cost/bit      | Much higher                           | Lower                                  |  [scribd](https://www.scribd.com/document/926702258/q21)

    - Cache is organized in **lines** (e.g., 64 bytes), with policies like direct-mapped, set-associative, fully associative, plus LRU/PLRU replacement. [dev](https://dev.to/sachin_tolay_052a7e539e57/understanding-cpu-cache-organization-and-structure-4o1o)
    - When CPU needs data: check L1 → L2 → L3 → RAM; on miss, bring a line from lower level up, exploiting spatial + temporal locality. [scribd](https://www.scribd.com/document/926702258/q21)

    ### Cache / RAM in scalable systems

    - In a single machine, you can add **software caches** (e.g., in-process LRU map) which conceptually act like another level above RAM.  
    - In distributed systems, you get:  
    - **In-memory caches**: Redis, Memcached, application caches, often on separate nodes but still RAM-based.  
    - **DB buffer cache**: DB’s own RAM-based cache of disk pages.  

    Hardware vs software:

    - CPU cache: invisible to app, managed by hardware. [dev](https://dev.to/sachin_tolay_052a7e539e57/understanding-cpu-cache-organization-and-structure-4o1o)
    - RAM: OS-managed; app sees a virtual address space. [scribd](https://www.scribd.com/document/926702258/q21)
    - Redis / Memcached: explicitly used by app as a fast key-value store to avoid DB hits.

    Question for you: If your service suddenly becomes read-heavy, which cache layer would you introduce first (DB-level cache, Redis, app in-memory cache), and what main tradeoff would you be thinking about?

    ***

    If you share what you’re preparing for (e.g., interviews vs building a specific system), this can be tuned to that context and we can drill into any one of these (e.g., detailed B+ tree layout or concrete replication configs).
# Quwst-2.
    Suggest scenarios, possibilities and explain below terms:
    1. partial network outages
    2. disconnected servers
    3. nodes become isolated but remain functional

    My name is GitHub Copilot and I am using Gemini 3 Flash Preview.

    In the context of distributed systems and leader election, these terms describe various failure modes that challenge consensus:

    1. Partial Network Outages
    Explanation: A state where communication between nodes is inconsistent. Unlike a total blackout, some packets might get through, or communication might be unidirectional (Node A can hear Node B, but Node B cannot hear Node A).

    Scenario: A faulty network switch or router starts dropping 50% of packets, or a misconfigured firewall rule blocks specific ports while leaving others open.
    Possibility: This can lead to "flapping," where a leader is repeatedly marked as down and then up, causing constant re-elections and system instability (high churn). - How to fix this?
    2. Disconnected Servers
    Explanation: A total loss of connectivity between one or more servers and the rest of the cluster. The server is still running, but it cannot send or receive any heartbeat signals or data.

    Scenario: A network interface card (NIC) hardware failure or a maintenance worker accidentally unplugging a network cable.
    Possibility: If the disconnected server was the leader, the remaining nodes will detect the loss of heartbeats and initiate a new election. If the server was a follower, it simply becomes unreachable until it is reconnected.
    3. Nodes Become Isolated but Remain Functional
    Explanation: Also known as a Network Partition. A subset of nodes can communicate with each other but are cut off from the rest of the cluster. These nodes continue to execute code and respond to local requests.

    Scenario: In a multi-region deployment, the undersea cable or fiber link between regions is severed. Region A nodes can talk to each other, and Region B nodes can talk to each other, but Region A cannot talk to Region B.
    Possibility: Split-Brain Scenario. If the isolation isn't handled by a consensus algorithm (like Raft or Paxos), both sides of the partition might elect their own leader. This can result in data corruption as two different "leaders" process conflicting writes simultaneously. Consensus algorithms prevent this by requiring a "quorum" (majority) to elect a leader or commit data.
# Quest-3.
- Question: Applications/Websites use payment gateways - do they poll to confirm payment received? Share different approaches used, possible to use and reasons behind their architecture. 

- ### Answer:
    While some applications use polling, modern architectures primarily rely on Webhooks (Push notifications) combined with polling as a fallback. Here are the common approaches and the architectural reasoning behind them:
    1. Webhooks (Asynchronous Server-to-Server)
    The payment gateway sends an HTTP POST request to the merchant's server as soon as the payment status changes (e.g., payment.succeeded).

    Architecture: The gateway acts as a client, and the application acts as a listener.
    Reasons:
    Efficiency: Eliminates unnecessary network traffic compared to repeated polling.
    Real-time: Updates the system-of-record immediately upon completion.
    Reliability: Most gateways (Stripe, PayPal, Razorpay) implement retry logic if the merchant's server is down.
    2. Polling (Synchronous/Asynchronous Pull)
    The application repeatedly asks the gateway API for the status of a specific transaction ID.

    Approaches:
    Client-side Polling: The browser/mobile app pings the merchant's backend, which then pings the gateway.
    Server-side Polling: A background worker on the merchant's server checks the status every few seconds.
    Reasons:
    Fallback: Essential if a webhook is missed or the network fails during the push notification.
    UX: Used on "Processing..." screens to update the UI the moment the server confirms the payment.
    3. Synchronous Redirects (Front-end Callback)
    The user is redirected to a "Success" or "Failure" URL with parameters (e.g., ?status=success&txn_id=123).

    Architecture: Relies on the user's browser to pass the state back to the application.
    Reasons:
    User Experience: Provides immediate visual feedback to the customer.
    Insecurity: This is not used for updating the database or fulfilling orders because users can manipulate URL parameters or close the browser before the redirect finishes.
    4. Comparison Summary
    Feature	Webhooks	Polling	Redirects
    Direction	Gateway 
    →
    → App	App 
    →
    → Gateway	Browser 
    →
    → App
    Reliability	High (Retry logic)	Medium (Resource heavy)	Low (User dependent)
    Primary Use	Order fulfillment	UI updates / Fallback	Immediate UX feedback
    Recommended Architecture:
    Use Webhooks as the primary mechanism for business logic (updating databases, shipping items) and Client-side Polling solely to transition the UI from a "Pending" state to a "Success" state. Always implement a Reconciliation Job (periodic polling) to catch any transactions that missed both webhooks and redirects.

# Quest-4.
    Perplexity's System Design Thread:
    In-memory vs RAM vs Cache vs Memory vs ROM
- ### Answer:
    Here’s a compact, concept-clarifying view of all five terms.

    ***
    ## 1. “Memory” (generic)
    Informal umbrella word that can mean:

    - Any place that stores bits: registers, cache, RAM, SSD, HDD.
    - In CS theory: an abstract addressable store.

    When people say “your computer’s memory” in day-to-day speech, they almost always mean **RAM**, not SSD/HDD. [linkedin](https://www.linkedin.com/posts/pawan-kumar-581b5b246_computerbasics-csstudents-techconcepts-activity-7420783344801480704-GHaJ)

    ***
    ## 2. RAM (main memory)
    **RAM = Random Access Memory**

    - **What it is:** Primary working memory where the CPU sees programs and data while they’re running. [linkedin](https://www.linkedin.com/posts/pawan-kumar-581b5b246_computerbasics-csstudents-techconcepts-activity-7420783344801480704-GHaJ)
    - **Properties:**
    - **Volatile:** Loses content when power is off. [youtube](https://www.youtube.com/watch?v=xfIF_H1uLEg)
    - **Speed:** Much faster than SSD/HDD, slower than CPU cache. [youtube](https://www.youtube.com/watch?v=p_7x2uFM2cM)
    - **Capacity:** GB–TB range, much larger than cache. [educatly](https://www.educatly.com/blog/818/primary-storage-explained-ram-cache-registers)
    - **Use in systems design:**  
    - DB buffer caches, in-memory key–value stores, JVM heaps, Golang heaps all live in RAM.
    - When you say “load it into memory” at system-design scale, you usually mean “keep hot working set in RAM”.

    ***
    ## 3. Cache (CPU cache, memory cache)
    **Cache = a small, fast copy of hotter data, sitting near the consumer.**
    ### Hardware (CPU) cache
    - **Location:** On-die or very close to CPU (L1, L2, L3). [educatly](https://www.educatly.com/blog/818/primary-storage-explained-ram-cache-registers)
    - **Properties:**
    - **Very small:** KB–tens of MB. [educatly](https://www.educatly.com/blog/818/primary-storage-explained-ram-cache-registers)
    - **Very fast:** Faster than RAM; CPU always checks cache before RAM. [youtube](https://www.youtube.com/watch?v=p_7x2uFM2cM)
    - **Volatile.** [youtube](https://www.youtube.com/watch?v=xfIF_H1uLEg)
    - **Role:** Keeps recently/frequently used instructions and data so the CPU doesn’t wait on slower RAM every time. [educatly](https://www.educatly.com/blog/818/primary-storage-explained-ram-cache-registers)
    ### Software / system caches
    - **Examples:** OS page cache, DB buffer cache, Redis/Memcached, HTTP reverse-proxy cache.
    - They conceptually mirror CPU cache:
    - Smaller, faster tier (RAM in a cache node) in front of a slower tier (disk, remote DB).
    - Store **redundant copies**; original remains in backing store.

    ***
    ## 4. ROM
    **ROM = Read-Only Memory**

    - **What it is:** Non-volatile memory that retains contents without power. [youtube](https://www.youtube.com/watch?v=xfIF_H1uLEg)
    - **Properties:**
    - Traditionally **read-only** (now often flash/EEPROM, so reprogrammable but not frequently).
    - Slower than RAM, but data is permanent firmware. [youtube](https://www.youtube.com/watch?v=8QKleMxvOiU)
    - **Use:**  
    - BIOS/UEFI firmware, microcontroller boot code, embedded device firmware.

    Think: **“code that must be there on power-on and must not vanish”**.

    ***
    ## 5. “In-memory” (in-memory DB / computing)
    “In-memory” is about **where your data/processing live in the hierarchy**.

    - **In-memory database:** DB that keeps most/all operational data directly in main memory (RAM) instead of disk. [gridgain](https://www.gridgain.com/resources/glossary/in-memory-computing-platform/in-memory-database)
    - Examples: Redis (as DB), SAP HANA, MemSQL/SingleStore, Exasol. [exasol](https://www.exasol.com/glossary/in-memory-database-definition/)
    - **Reason:** RAM access is orders of magnitude faster than disk; queries avoid disk seeks and complex I/O paths. [en.wikipedia](https://en.wikipedia.org/wiki/In-memory_database)
    - **In-memory cache:** Redis/Memcached keeping hot data in RAM on a separate tier.
    - **In-memory computing platform:** Whole compute pipeline (state, intermediate results) is kept in RAM across nodes (e.g., Spark with enough executors and memory).

    So:

    - **“In-memory” = “resides in RAM rather than disk”**, at the software/system level. [gridgain](https://www.gridgain.com/resources/glossary/in-memory-computing-platform/in-memory-database)
    - RAM is the **hardware**; “in-memory” is a **design choice** about using RAM as the primary backing store.

    ***
    ## 6. Putting it all together (mental model)
    Hierarchical view from CPU outward:

    1. **Registers** – inside CPU, fastest, tiny.
    2. **CPU cache (L1/L2/L3)** – on-die, very fast, small, volatile. [educatly](https://www.educatly.com/blog/818/primary-storage-explained-ram-cache-registers)
    3. **RAM (main memory)** – on motherboard, fast, larger, volatile. [linkedin](https://www.linkedin.com/posts/pawan-kumar-581b5b246_computerbasics-csstudents-techconcepts-activity-7420783344801480704-GHaJ)
    4. **“In-memory” systems** – software that deliberately keeps its working set in RAM instead of disk. [gridgain](https://www.gridgain.com/resources/glossary/in-memory-computing-platform/in-memory-database)
    5. **Storage (SSD/HDD)** – persistent, slowest, huge.
    6. **ROM / Flash** – persistent firmware storage, rarely written. [youtube](https://www.youtube.com/watch?v=xfIF_H1uLEg)

    You can think of:

    - **Cache vs RAM:** both volatile; cache is *smaller and closer* to the CPU, RAM is *bigger and a bit farther*. [youtube](https://www.youtube.com/watch?v=p_7x2uFM2cM)
    - **RAM vs in-memory:** RAM is the medium; “in-memory” is *choosing* to put your data structures or DB there instead of disk. [en.wikipedia](https://en.wikipedia.org/wiki/In-memory_database)
    - **Memory vs ROM:** generic vs specific; ROM is a specific non-volatile memory type for firmware.

    ***

    To make this more system-design-specific: which of these do you most want to drill into next—CPU/RAM/cache hierarchy as it impacts latency and throughput, or “in-memory” architectures like Redis/SAP HANA/SingleStore?