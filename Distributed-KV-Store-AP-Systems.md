# Distributed Key-Value Stores: AP Systems & Storage Engines
*Comprehensive SDE-2 Level Notes on Leaderless Architectures (Dynamo/Cassandra) and Spanner.*

## 1. Request Flow and Quorum (Leaderless Architecture)
In a highly available (AP) leaderless system, there is no single Master node. 
* **The Coordinator:** The Load Balancer routes a client request to *any* node. That node becomes the **Coordinator** for the request.
* **Consistent Hashing:** The Coordinator hashes the key to find the $N$ replica nodes that own the data (skipping nodes to ensure cross-rack/cross-AZ placement).
* **Parallel Writes:** The Coordinator does *not* write locally and replicate asynchronously. It broadcasts the write to all $N$ replicas in parallel.
* **Quorum ($W$ and $R$):** The system returns "Success" as soon as a Write Quorum ($W$) of nodes acknowledge the write. (e.g., $N=3, W=2$).

### Multi-Region Deployments
* Cross-region writes (e.g., India to NAM) directly over the internet are too slow. 
* Instead, DNS Geo-Routing points users to a local Load Balancer and a local cluster. 
* Datacenters replicate to each other asynchronously in the background.

---

## 2. Handling Node Failures (Eventual Consistency)
If nodes crash or miss writes, the system relies on three mechanisms to self-heal and achieve Eventual Consistency.

1. **Hinted Handoff (Immediate Write Path):**
   * If a target node is down, the system uses a **Sloppy Quorum**. The Coordinator writes to a healthy non-owner node (the next one on the ring) and attaches a "hint" (e.g., *"This belongs to Node A"*). 
   * When Node A recovers, the temporary node hands the data over and deletes its local copy.
2. **Read Repair (On the Read Path):**
   * When a client reads data, the Coordinator fetches from $R$ nodes. If it detects stale data (via timestamps/Vector Clocks), it returns the newest data to the client and asynchronously overwrites the stale nodes in the background.
3. **Anti-Entropy via Merkle Trees (Background Sync):**
   * For "cold" data that isn't actively read, replicas continuously gossip.
   * They compare **Merkle Trees** (hash trees) of their data. If root hashes mismatch, they traverse the tree to find the exact differing keys and exchange only the missing data.

---

## 3. Storage Engine: LSM-Trees & SSTables
Traditional RDBMS (Oracle/B-Trees) struggle with massive write throughput due to Random I/O and page-split locks. AP systems use **Log-Structured Merge-Trees (LSM-Trees)** for sequential I/O.

* **MemTable (RAM):** Writes go into an in-memory data structure (like a Skip List or Red-Black Tree). It keeps keys perfectly sorted.
* **Commit Log (Disk):** Writes are appended to a log strictly for crash recovery.
* **SSTable (Disk):** When the MemTable is full, it flushes to disk as a **Sorted String Table**.
  * **Immutable:** SSTables are never modified.
  * **Updates/Deletes:** Handled by writing a new timestamped record or a **Tombstone** (deletion marker).
* **Compaction:** A background process periodically merges old SSTables, discarding overwritten data and tombstones to reclaim disk space and speed up reads.

---

## 4. Bloom Filters (Fast Reads)
To prevent the database from scanning dozens of SSTables on disk for a single read, an in-memory **Bloom Filter** is attached to every SSTable.

* **Mechanism:** It uses a bit array and $k$ hash functions. 
* **Outcomes:** It never returns a false negative. It returns either **"Definitely Not"** (0 disk reads wasted) or **"Possibly Yes"** (triggering a disk read).
* **Math & Optimization:** The size of the array ($m$) and number of hashes ($k$) are calculated dynamically based on the target false-positive rate ($p$) and number of items ($n$). 
* A perfectly optimized Bloom filter is exactly 50% full of `1`s, mathematically minimizing false positives.

---

## 5. Conflict Resolution & Vector Clocks
Because there is no Master node, two clients can write to two different Coordinators simultaneously, causing conflicts. Wall-clock timestamps are unreliable due to **Clock Drift**.

* **Vector Clocks:** Track causality using an array of counters `[Node: Counter]`. (e.g., `[A:2, B:1]`).
* **Coordinator Minting:** The Coordinator mints the new clock and broadcasts it. Replicas passively accept it.
* **Context Passing:** Clients must perform a Read-Modify-Write. They read the data, get the Vector Clock (Context), and send it back with the update so the new Coordinator knows the causal history.
* **Conflict Detection:** If two clocks are compared and neither strictly dominates the other (e.g., `[A:2]` vs `[A:1, C:1]`), it is a concurrent conflict. The database returns both "siblings" to the client application to merge.
* *Note: Modern Cassandra abandoned Vector Clocks due to complexity (e.g., network retries creating false siblings) and now uses Last-Write-Wins (LWW).*

---

## 6. Google Spanner & TrueTime (CP Systems)
Google Spanner provides global strict consistency (CP) without replication mismatches, bypassing the flaws of eventual consistency.

* **The Problem:** Clock drift (quartz oscillators ticking at different speeds) destroys causality in distributed systems. 
* **TrueTime API:** Backed by GPS and atomic clocks, TrueTime returns an uncertainty window `[earliest, latest]` instead of an absolute time (max drift ~7ms).
* **Commit Wait:** The Spanner Leader assigns the `latest` timestamp to a write, but **waits out the 7ms uncertainty window** before telling the client "Success".
* **Result:** This mathematically guarantees that any subsequent transaction triggered by a human will have a strictly greater timestamp. Causality is preserved globally.
* **Paxos & Lock-Free Reads:** Replicas use Paxos to agree on the exact TrueTime sequence. Because the history is perfectly ordered globally, clients can perform lock-free read-only transactions by simply querying the database state at a specific TrueTime in the past.