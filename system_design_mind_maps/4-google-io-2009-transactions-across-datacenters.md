Model used: **Claude Sonnet 4.6** (Anthropic), deep system-design mode.

Below is a detailed Markdown “note” you can save as something like  
`google-io-2009-transactions-across-datacenters.md`.

***

# Google I/O 2009 — Transactions Across Datacenters (Ryan Barrett)  
Multihoming, Consistency, Transactions, and Modern Approaches (2026 View)

Sources: Google I/O 2009 talk “Transactions Across Datacenters (and Other Weekend Projects)” by Ryan Barrett, plus modern system-design practice and your consistency-patterns notes. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

***

## 1. Problem Framing: Multihoming Structured Storage

### 1.1 What is “multihoming”?

- **Multihoming** = running a service from **multiple datacenters** so it can survive:
  - Catastrophic failures (power loss, fire, DC outage).
  - Expected failures (router maintenance, cooling/power issues).
  - Latency constraints (placing data closer to users). [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- Focus of the talk: **read–write “structured data”** (user data with expectations of durability, correctness, and fresh reads), not static content or pure batch processing. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Why this is hard:

- Inside a DC: high bandwidth, low latency, cheap internal traffic.
- Across DCs: slower, more expensive, limited bandwidth, and **high latency (speed-of-light)** becomes dominant. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- As soon as you accept **writes in more than one location**, you must solve:
  - Consistency of state.
  - Conflict resolution.
  - Transactional invariants across sites.

***

## 2. Consistency Levels (Weak, Eventual, Strong)

The talk uses a practical three-level model; this lines up with your consistency-patterns section. [hazelcast](https://hazelcast.com/foundations/distributed-computing/sharding/)

### 2.1 Weak consistency

- Typical for **caches**:
  - You write; later reads **may** see it, may not.
  - No guarantees that a read after write returns the new value. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- Examples:
  - Memcache on App Engine (best-effort cache).
  - VoIP and online games: dropping/losing updates is acceptable if the stream continues smoothly.

When is it acceptable?

- When the data is **not authoritative** and can be recomputed or refetched.
- When **freshness is less important than latency**.

### 2.2 Eventual consistency

- Guarantee: if no new writes happen, **all replicas will eventually converge** to the same value.
- Reads shortly after a write may be stale, but given time, everyone sees the write. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- Examples:
  - Email delivery (SMTP).
  - DNS propagation.
  - Many 2000s-era cloud stores: Amazon S3, SimpleDB, SQS were classic examples. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

### 2.3 Strong consistency

- Guarantee: once a write is acknowledged, **all subsequent reads see that write**. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- This is essentially **linearizable** behavior from the client perspective.
- Examples:
  - File systems.
  - Relational databases (single-node view).
  - App Engine Datastore (as of the talk) and Azure Tables were cited as strongly consistent. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Why strong consistency is attractive:

- **Simplest mental model** for developers: no need to reason about stale reads, write visibility, or weird anomalies.
- But it becomes **hard and expensive** across datacenters because of latency and coordination.

***

## 3. Transactions and ACID (in this Context)

### 3.1 What is a transaction here?

Classic bank example (transfer from A to B): [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

- Two operations:
  - Subtract from A.
  - Add to B.
- With concurrent operations and possible failures:
  - You must not “create” or “destroy” money.
  - Invariants must hold (e.g., total balance is preserved).

Properties (ACID):

- **Atomicity** – all or nothing.
- **Consistency** – invariants preserved.
- **Isolation** – concurrent transactions don’t interfere.
- **Durability** – once committed, stays committed.

In a **single datacenter**, ACID is already nontrivial but well understood.  
Across datacenters, transactions interact with **latency, partitions, and coordination**.

***

## 4. Why (and Why Not) Multihome

### 4.1 Reasons to multihome

1. **Catastrophic failure tolerance**  
   - DC fire, power loss, or even “DC fell in the ocean” thought experiment. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
   - You don’t want prolonged downtime or permanent data loss.

2. **Expected failures and maintenance**
   - Big routers, cooling, backbone links all need maintenance and occasionally fail.
   - Having another DC lets you keep serving while one is down.

3. **Geolocation / proximity to users**
   - Users in India, China, or Australia suffer large RTTs if everything is in a single US DC. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
   - CDNs solve static/edge caching; structured data still needs multi-DC strategies.

### 4.2 Reasons not to multihome (or to be cautious)

- Cross-DC bandwidth is smaller and more expensive.
- Latency is significantly higher (speed-of-light + queuing/routing).
- Multihoming adds substantial **complexity in consistency, failover, schema, and ops**. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- Many systems instead “bunkerize” a single DC: heroic redundancy for power, cooling, uplinks (but still single region). [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

***

## 5. Degrees of Multihoming

The talk sets up a spectrum, which maps nicely onto modern patterns. [hazelcast](https://hazelcast.com/foundations/distributed-computing/sharding/)

### 5.1 None (single DC, bunkerized)

- All reads and writes served from one DC.
- Data is backed up, possibly to offsite storage, but there is no live second DC.
- Upside: simple; best latency and throughput.
- Downside: if DC fails, you’re **down and possibly out** (e.g., Twitter/FriendFeed outage from SVColo power failure).  

This is **“CA” under CAP only if you assume no partitions**, which is unrealistic long-term.

### 5.2 Some: master + read-only replicas in other DCs

Pattern:

- One DC is the **master**:
  - All writes go here.
  - Reads can be served from master and/or read replicas in other DCs.
- Other DCs are **slaves/replicas**:
  - Receive **asynchronous replication** of the master’s log. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Properties:

- Latency for writes roughly same as single-DC (no cross-DC sync on write path).
- Reads can be:
  - Served closer to users (read replicas).
  - Slightly stale (replication lag).
- Failure:
  - If master DC fails:
    - You may promote a replica to master (manual or automated failover).
    - There’s usually a **small window of unreplicated writes** that may be lost or require repair. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

This is functionally **single-master, async replication** in modern terms.

### 5.3 Full: multi-master, fully distributed across DCs

Pattern:

- **Writes accepted in multiple DCs**.
- Replication runs between DCs; system must:
  - Reconcile conflicts.
  - Or impose a distributed total/partial order on updates.
- Ideally, all DCs can serve reads and writes with some consistency guarantee.

This is the “holy grail” that the talk explores via 4 key techniques. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

***

## 6. Techniques for Multihoming Across Datacenters

The talk walks through five main approaches and evaluates them on:

- Consistency/transaction guarantees.
- Latency.
- Throughput.
- Data loss risk.
- Failover behavior. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

### 6.1 Technique #1 – Backups

**Idea:**

- Periodically copy all data to another location (tape, remote storage, another DC).
- Mostly **offline**; doesn’t affect online traffic much.

Pros:

- No impact on online latency or throughput.
- Simple and essential for **disaster recovery**. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Cons:

- Possible large **RPO** (Recovery Point Objective): all data since last backup can be lost.
- Consistency:
  - Naïve scanning backup gets inconsistent snapshot (start of backup sees older state than end).
  - Correct, transactional snapshotting is possible but non-trivial.

Use today:

- Still fundamental as a **baseline**: offline backups + WAL shipping + snapshotting.
- Not sufficient for **low RPO or active-active** multihoming.

### 6.2 Technique #2 – Master/Slave (Single Master, Async Replication)

**Idea:**

- One DC is the authoritative **master** for writes.
- Others are **slaves/replicas** that receive async replication of an ordered log of updates. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Pros:

- Does not significantly increase write latency (replication is async).
- Can provide:
  - Strong consistency **within a DC**.
  - Transactional semantics preserved if replication replays **committed log entries**. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- Failover:
  - When master fails:
    - You promote a replica to master.
    - Reads can still be served from replicas; writes resume once failover completes.

Cons:

- Small window for data loss (writes that were committed locally but not yet replicated).
- Slaves may be behind; cross-region reads can be stale.
- During master failover, writes might be disabled or limited.

App Engine Datastore in 2009:

- Chose this approach for multi-DC: **master-slave replication** at the level of per-entity-group commit logs. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- They wanted strong consistency per entity group but **could not afford the latency** of cross-DC sync on every write.

Modern comparison:

- This is still widely used:
  - Primary/replica setups in MySQL/Postgres.
  - Many managed cloud RDBMS/NoSQL offerings.

### 6.3 Technique #3 – Multi-master replication

**Idea:**

- Allow **writes in multiple DCs**.
- Replicate updates between DCs and **merge** them later.
- Use:
  - Versioning / timestamps / vector clocks.
  - Conflict resolution policies (last-write-wins, app-specific merge). [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Pros:

- All DCs can accept writes; better write-side locality and availability.
- Good for **eventual consistency** use cases.

Cons:

- At best you get **eventual consistency**, not global strong consistency.
- Cross-entity or cross-partition transactions are essentially impossible without additional mechanisms.
- Complexity in conflict detection and resolution:
  - E.g., “double-spend” problems if two DCs both decrement the same balance independently.

App Engine view in 2009:

- Rejected multi-master for the Datastore, because they **wanted strong consistency** and found eventual consistency too hard for app developers to reason about. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Modern analogy:

- Dynamo-style systems (Cassandra, Riak, DynamoDB in some modes).
- CRDT-based systems for convergent structures.

### 6.4 Technique #4 – Two-Phase Commit (2PC) Across DCs

**Idea:**

- Use a **coordinator** to run two-phase commit across DCs:
  - Phase 1 (prepare): ask all participants if they can commit.
  - Phase 2 (commit/abort): if all are OK, tell everyone to commit; else abort. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Pros:

- Gives **atomic, strongly consistent cross-site transactions**:
  - Each transaction either commits on all DCs or none.
- Good for high-value, strictly consistent workloads (e.g., cross-DC trading systems, like the NASDAQ anecdote). [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Cons:

- Coordinator becomes a **bottleneck**:
  - All 2PC transactions for that shard go through one node.
  - Serialization point reduces throughput.
- **High latency**:
  - At least two cross-DC RTTs in the critical path of a write.
  - Can easily push writes into **150–250 ms+** range, which is an order of magnitude slower than local writes. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- Blocking: if coordinator fails at wrong time, participants can be in uncertain “in-doubt” state (3PC alleviates some but adds complexity).

App Engine view:

- Considered but rejected 2PC for the datastore because:
  - It would make writes dramatically slower compared to single-DC DBs.
  - Their competitive baseline was already local DBs with single-digit millisecond writes. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

### 6.5 Technique #5 – Paxos (Distributed Consensus)

**Idea:**

- Use **Paxos** (or similar consensus protocol) to replicate a log across nodes and DCs:
  - Consensus on each log slot ensures a single authoritative ordered log for updates. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- Fully distributed (no single permanent master for each value), majority-quorum based.

Pros:

- Strong consistency (linearizable log if used correctly).
- Fault-tolerant to node and network failures as long as a majority is reachable.
- No single coordinator per transaction in principle; can pipeline and parallelize.

Cons:

- Still needs **cross-DC majority communication**:
  - At least one (often effectively two) RTTs across DC boundaries per consensus instance.
  - Similar write latency to 2PC in practice for geo-distributed scenarios.
- Non-trivial to implement correctly, reason about, and tune.

App Engine view:

- Google heavily used Paxos internally (Chubby lock service, etc.). [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)
- They experimented with Paxos across DCs for the datastore, but:
  - Couldn’t hit latency targets (would raise write latency from ~30–50 ms to ~150 ms+).
  - Ultimately decided to use Paxos internally for **coordination and control-plane tasks**, not as the main, per-write data plane. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Modern analogy (2026):

- Many systems now use **Raft (Paxos-like)** as the primary replication mechanism (e.g., etcd, Consul, CockroachDB, Yugabyte, Spanner’s internal Paxos, etc.). [linkedin](https://www.linkedin.com/pulse/beating-heart-kubernetes-deep-dive-raft-consensus-etcd-jisan-ahmed-bwcjc)
- Latency is mitigated by:
  - Focusing writes to a **regional majority** (multi-region but not globally arbitrary).
  - Providing **read-local but bounded-stale** options in secondary regions.

***

## 7. Summary of Trade-offs (2009 Perspective)

The talk ends with a quasi “scorecard” (green/yellow/red) across techniques. [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

Very roughly:

- **Backups**
  - + Latency/throughput: great (no online impact).
  - – Data loss window: large.
  - – Failover/availability: poor.

- **Master/Slave**
  - + Strong consistency in master DC.
  - + Good latency, good throughput.
  - ± Small data-loss window.
  - ± Failover requires promotion; some downtime or read-only periods.

- **Multi-master**
  - + High availability and write locality (all DCs accept writes).
  - + Good throughput.
  - – Only eventual consistency.
  - – Complex conflict resolution; transactional invariants hard.

- **2PC across DCs**
  - + Strong consistency, cross-DC atomic transactions.
  - – Poor write latency (2+ DC RTTs).
  - – Throughput bottleneck at coordinator.

- **Paxos**
  - + Strong consistency.
  - + Better throughput than 2PC (no single per-tx coordinator).
  - – Still high latency for geo-distribution.
  - – Complexity in implementation and operational tuning.

Key conclusion in 2009:

> “There is no silver bullet; life is tradeoffs. For the App Engine Datastore we chose master/slave across datacenters as the sweet spot between consistency, latency, and operational complexity.” [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

They also hinted at a future where **developers could choose per-write**:

- “Strong writes” (synchronous, cross-DC consensus) vs  
- “Fast writes” (local, async replication, potentially stale remote reads). [stackoverflow](https://stackoverflow.com/questions/35387774/is-zookeeper-always-consistent-in-terms-of-cap-theorem)

This is exactly what modern “tunable consistency” APIs now offer.

***

## 8. How the Landscape Evolved (2010s–2026)

Since 2009, several significant shifts have made multi-DC consistency more approachable.

### 8.1 Consensus protocols in practice: Paxos → Raft → production tooling

- **Raft** (2013) made consensus easier to understand and implement; widely adopted in etcd, Consul, TiKV, CockroachDB, many internal services. [youngju](https://www.youngju.dev/blog/etcd/etcd_architecture_internals.en)
- Standardized leader-based consensus + log replication is now baseline for:
  - Config and coordination (etcd, ZooKeeper replacements).
  - Strongly-consistent databases (e.g., CockroachDB, YugabyteDB, Spanner’s Paxos groups).

This means **“Paxos everywhere”** is now a reality, not just a Google internal footnote.

### 8.2 Global databases with bounded-staleness and multi-region replicas

Examples (high level):

- **Google Spanner**:  
  - True multi-DC, globally-distributed SQL with external consistency (strict serialization). [systemdesignhandbook](https://www.systemdesignhandbook.com/guides/zookeeper-system-design/)
  - Uses Paxos for replication and TrueTime for global ordering; exposes **per-transaction knobs** for anti-latency vs global consistency.
- **CockroachDB, YugabyteDB**:
  - Open-source, Spanner-inspired systems using Raft.
  - Provide multi-region table placement, **per-table or per-query** consistency/latency trade-offs.

In CAP terms:

- Strong **CP** behavior for critical data.
- Various **read-local/lagged replica** features for better latency on read-heavy or less-critical workloads.

### 8.3 Cloud primitives: regional vs global services

- Managed services from hyperscalers now expose:
  - **Regional** strongly-consistent endpoints (e.g., per-region leaders).
  - **Global** read replicas with bounded staleness or eventual consistency.
- Developers can choose:
  - Local-region, **low-latency strong writes**.
  - Multi-region, **slightly higher latency but globally consistent** writes.
  - Or **AP-style** services (e.g., Dynamo-like key-value stores) for high availability and write throughput at the cost of consistency.

This largely realizes the “per-write/per-entity” toggle Ryan Barrett foreshadowed.

### 8.4 Better consistency patterns surfaced in teaching materials

Your system-design primer’s **consistency patterns** section generalizes the talk’s core themes: [hazelcast](https://hazelcast.com/foundations/distributed-computing/sharding/)

- Weak vs eventual vs strong consistency.
- Read-your-writes, monotonic reads, etc.
- AP vs CP trade-offs under CAP.

These patterns are now standard vocabulary during design discussions and interviews.

***

## 9. How to Apply These Ideas in Modern System Design

When you design a system in 2026, you can treat the talk as a **conceptual baseline** and then plug in modern tooling.

### 9.1 Decision steps

1. **Classify your data and operations**:
   - Critical, cross-account invariants (money, inventory) → require **strong consistency, often CP**.
   - User-facing, “soft-state” data (feeds, likes, analytics) → can tolerate **staleness or eventual consistency**.

2. **Pick your multihoming level per subsystem**:
   - For critical data:
     - Use **regional CP database + async multi-region replicas**, or
     - Use **global CP database** (Spanner/Cockroach) if you accept higher write latency.
   - For soft data:
     - Use **multi-master or AP** systems (Cassandra, Dynamo-like, CRDT-based caches).

3. **Choose technique mapping**:
   - Backups are always present for DR.
   - Master/slave (or leader/follower) for most OLTP data.
   - Paxos/Raft as underlying replication for “CP” stores.
   - Rarely use full 2PC across DCs; instead:
     - Use smaller consensus/transaction scopes.
     - Model cross-DC invariants with **compensation and sagas** rather than single atomic transactions.

4. **Expose the trade-off explicitly to callers**:
   - APIs with options like:
     - `read_consistency = STRONG | BOUNDED_STALENESS | EVENTUAL`
     - `write_mode = SYNCHRONOUS | ASYNCHRONOUS`
   - This mirrors the “future direction” from the 2009 talk.

***

To practice: pick one concrete system you know (say, a modern multi-region Postgres + read-replicas + Redis + Kafka setup), and try to label which **multihoming level and technique** each component is effectively using: “some” (master + replicas), “full” (multi-master/eventual), or “none” (single-region only).