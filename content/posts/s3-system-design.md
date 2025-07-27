+++
title = "S3 system design"
date = "2025-07-27"
summary = "Detailed System Design: A Global, Scalable Object Storage Service (S3-like)"
draft = false
+++

### **Detailed System Design: A Global, Scalable Object Storage Service (S3-like)**

As a candidate for a Staff Engineer position, my goal is to present a design that is not only functional but also robust, scalable, and operationally sound. This design for a global object storage service is built on proven principles of distributed systems to meet extreme requirements for durability, availability, and scale.

#### **1. Core Principles & Requirements**

First, let's restate our guiding principles and goals.

- **Guiding Principles:**

  1.  **Separate Metadata from Data:** Scale and optimize each path independently.
  2.  **Design for Failure:** Assume commodity hardware will fail; build resilience in software.
  3.  **Scale Horizontally:** Achieve near-infinite scale by adding more commodity servers.
  4.  **Global System, Regional Performance:** Provide a single global namespace but with low-latency access via regional endpoints.

- **Key Requirements:**
  - **Durability:** 99.999999999% (11 nines).
  - **Availability:** 99.99% within a region.
  - **Scale:** Exabytes of storage, trillions of objects, millions of RPS.
  - **Consistency:** Strong read-after-write for new objects; eventual for overwrites/deletes.

---

#### **2. The Global Architecture: A Multi-Layered View**

Our system is composed of several specialized layers, each with a distinct responsibility.

```
+-----------+
|   User    |
+-----+-----+
      |
      v
+-----+----------------------+
|   Layer 1: The Global Edge   | (Anycast DNS, API Gateway PoPs)
|   (Handles initial connection, |
|    routing, and caching)     |
+-------------+--------------+
              |
              | (Internal, Proxied Request)
              v
+-------------+--------------+
|   Layer 2: The Regional Core | (The primary S3 service in a specific region)
|                              |
|   [API Gateway] <------> [Metadata Service] <------> [Storage Service]
|                              |
|                              v
|                       [Background Services]
|   (Replicators, Janitors, etc.)
+------------------------------+
```

---

#### **3. Deep Dive into Layers and Components**

##### **Layer 1: The Global Edge Network**

The user's first point of contact. Its goal is to provide a fast, secure, and unified entry point to our global service.

- **Anycast DNS:** We use a single global hostname (e.g., `s3.my-cloud.com`) that resolves to a single Anycast IP address. This IP is advertised from dozens of Points of Presence (PoPs) worldwide. Internet routing (BGP) automatically directs each user to the nearest PoP, ensuring low-latency for the initial connection.
- **Edge API Gateway:** Each PoP runs a fleet of our standard API Gateway servers. Their role here is to:
  1.  **Terminate TLS:** Perform the expensive cryptographic handshake close to the user.
  2.  **Authenticate:** Validate the user's credentials immediately.
  3.  **Route Intelligently:**
      - It contacts the **Global Bucket Service** (a tiny, globally-replicated database) to find out which region an object's bucket belongs to.
      - It then acts as a **secure proxy**, forwarding the request over our optimized private network backbone to the API Gateway in the correct destination region.
  4.  **Accelerate & Cache:** For uploads, it can receive the data quickly from the user and then handle the slower transfer to the destination region. For popular objects, it can act as a cache to serve reads directly from the edge.

##### **Layer 2: The Regional Core Service**

This is the complete, self-contained S3 implementation deployed in each geographic region (e.g., `us-east-1`, `eu-west-2`). All nodes within a region are spread across at least three Availability Zones (AZs) to survive a data center failure.

**a) Regional API Gateway**

- **Role:** The workhorse that orchestrates all read and write operations.
- **Logic:** It handles all application logic: authorization against bucket policies, chunking large files, and, most importantly, **performing on-the-fly Erasure Coding**. For a `PUT`, it calculates the data and parity shards. For a `GET`, it fetches the shards and reconstructs the object.
- **Transactionality:** It manages the atomic write process using our lightweight, non-locking transaction model:
  1.  **Prepare:** Creates a provisional, invisible metadata record.
  2.  **Write:** Streams data shards to the Storage Service.
  3.  **Commit:** Atomically flips the metadata record to a "committed" state, making the object visible.

**b) Metadata Service**

- **Role:** The authoritative, strongly consistent index of all objects within the region.
- **Architecture:** A **partitioned, replicated Key-Value store**.
  - **Partitioning:** We use **Consistent Hashing** on the object key to map each object to a specific partition. This allows us to scale horizontally by adding more partitions as the number of objects grows.
  - **Replication & Consistency:** Each partition is managed by a **3-node Partition Group using the Raft consensus algorithm**. These three nodes are placed in three different AZs. Raft ensures that all writes are replicated to a majority of nodes before being acknowledged, providing strong consistency and fault tolerance. Even if a whole AZ fails, the metadata service remains fully operational.
- **Data Model:** Stores a rich JSON blob for each object containing its size, checksums, permissions, and a list of all its data shard locations.

**c) Storage Service (Storage Nodes)**

- See section 4 below

**d) Background Services**

- **Janitor Process:** Cleans up failed writes by finding and deleting metadata records and data shards from transactions that were prepared but never committed.
- **Scrubber/Healer Process:** Continuously scans data shards in the background, verifies their checksums, and if a corrupt or missing shard is found, it automatically reconstructs it from the remaining shards and places it on a new, healthy node. This is what maintains our 11 nines of durability over time.
- **Replicator Service:** For optional cross-region replication, this service listens to an event stream of committed writes from another region and replicates them into its own regional core.

---

#### **4. Storage Service: The Pluggable Persistence Layer**

The Storage Service is the foundation of our system, responsible for the durable persistence of data shards. A key architectural decision is how much intelligence and responsibility we place on the individual storage nodes. Our design is flexible enough to support two powerful but distinct models. The choice between them depends on specific trade-offs between architectural simplicity, network traffic patterns, and operational philosophy.

The contract between the API Gateway and the Storage Service remains consistent: store this shard of data durably and retrieve it when asked. How the Storage Service achieves this durability internally is what differs.

---

##### **Model A: Smart Gateway + Application-Aware Storage (Erasure Coding at the Edge)**

This is the primary model discussed, emphasizing a strong separation of concerns.

- **Core Philosophy:** The API Gateway is responsible for data protection. The storage nodes are simple endpoints that store the final, already-encoded shards.
- **API Gateway's Role (Intelligent):**
  - Performs **Erasure Coding** on the fly, breaking an object into `k` data and `m` parity shards.
  - Orchestrates the transaction (`PREPARE`, `COMMIT`).
  - Selects `k+m` distinct Storage Nodes across multiple AZs and streams one unique shard to each.
- **Storage Node's Role (Simple, Application-Aware):**
  - Runs a specialized daemon that exposes a simple API: `put(shard_id, data)` and `get(shard_id)`.
  - It is **application-aware** in that it knows its purpose is to store shards for the object store. It optimizes its local disk layout for this (e.g., log-structured writes for small shards) and reports detailed health metrics.
  - **It does NOT perform replication.** Its durability guarantee for its single shard relies on its local hardware (e.g., local RAID is an option, but often eschewed in favor of software redundancy) and the overall system's healing process.
- **Write Path:** `API Gateway -> (Parallel Streams) -> 14 different Storage Nodes`
- **Trade-offs:**
  - **Pro:** Cleanest architectural separation. Complex logic is in the stateless, easy-to-scale gateway tier.
  - **Pro:** Gives the application layer full control over data placement and protection policies (e.g., choosing different EC schemes per bucket).
  - **Con:** Higher fan-out network traffic from the API Gateway during writes.
  - **Con:** Healing logic is centralized in a separate "Healer" service, which must coordinate reading from healthy nodes and writing to new ones.

---

##### **Model B: Simple Gateway + Smart Storage Cluster (Ceph-OSD Style)**

This model delegates the responsibility for data protection to an intelligent, self-managing storage cluster.

- **Core Philosophy:** The Storage Service itself is a complete, distributed storage system that provides a durable object-storage interface.
- **API Gateway's Role (Simpler):**
  - Acts primarily as a router and authenticator.
  - It does **not** perform erasure coding. It treats the entire Storage Service as a "black box" that stores objects durably.
  - It writes a large chunk of the object to a primary node within the storage cluster and lets the cluster handle the rest.
- **Storage Node's Role (Intelligent, Self-Managing "Ceph OSD-style"):**
  - Nodes are peers in a cluster and communicate extensively with each other.
  - **They handle replication and durability internally.** When a primary node receives data, it is responsible for replicating it (either via 3x replication or by coordinating an erasure coding scheme _with its peers_).
  - **They are self-healing.** If a node fails, the remaining peers detect this and automatically start a recovery process to regenerate the lost data and restore full redundancy, without needing a central "Healer" service.
  - Uses a data distribution algorithm like **CRUSH** to determine placement, minimizing the need for a central metadata lookup _for data placement_.
- **Write Path:** `API Gateway -> Primary Storage Node -> (Internal Replication/EC) -> Peer Storage Nodes`
- **Trade-offs:**
  - **Pro:** Simplifies the API Gateway.
  - **Pro:** Autonomous, self-managing storage plane is operationally robust and lessens the burden on higher-level services. Healing is decentralized and happens automatically.
  - **Con:** Blurs the architectural separation of concerns. The storage layer is now a highly complex, stateful distributed system.
  - **Con:** East-west (server-to-server) network traffic within the storage cluster can be very high due to replication and rebalancing.

---

#### **Conclusion: A Pluggable Choice**

Both models are valid and proven at scale. The choice is a significant architectural decision.

- **Choose Model A** for maximum flexibility, control at the application layer, and the cleanest separation of concerns. This is often favored by hyper-scalers building their own custom stack from the ground up.
- **Choose Model B** if you want to leverage a powerful, off-the-shelf, open-source storage system like Ceph as your backend. This provides a robust, self-managing foundation, trading some application-level control for operational autonomy.

Our system design is strong because the interface to the Storage Service is abstract enough that we can start with one model and even migrate to or integrate the other without changing the entire top-level architecture.

### **4. Data Models: The Blueprints of Our System**

The structure of our metadata is the contract that enables our services to work together. We have two primary data models: one for the globally-replicated bucket information and one for the regionally-stored object information.

#### **a) Object Metadata Model (Stored in the Regional Metadata Service)**

This is the most detailed record, containing everything needed to find, serve, and manage a single object. It's stored in the partitioned, Raft-based Key-Value store within each region.

- **Key:** `bucket_name/object_key` (e.g., "my-travel-photos/italy/rome.jpg")

- **Value (Example JSON representation):**

```json
{
  "version_id": "v3-a4e1b89f-3c8f-4b1a-9f2e-1d7c6b0a9e1d",
  "owner_id": "acct-123456789",
  "size_bytes": 104857600,
  "checksum": {
    "algorithm": "SHA256",
    "hash": "a1b2c3d4e5f6..."
  },
  "content_type": "image/jpeg",
  "creation_date": "2023-10-28T14:30:00Z",
  "storage_class": "STANDARD",
  "permissions_acl": {
    /* ... Standard ACL structure ... */
  },

  // --- Transactional State Fields ---
  "transaction_state": "COMMITTED", // Values: PREPARING, COMMITTED, DELETED
  "idempotency_token": "client-uuid-for-put-op-123", // Prevents duplicate writes on retry

  // --- Data Location and Durability Info ---
  "erasure_coding_info": {
    "scheme": "REED_SOLOMON_10_4", // 10 data shards, 4 parity shards
    "chunk_size_bytes": 10485760, // Each shard is 10 MiB
    "shards": [
      {
        "shard_id": "shard-xyz-01",
        "node_id": "storage-node-053",
        "az": "us-east-1a"
      },
      {
        "shard_id": "shard-xyz-02",
        "node_id": "storage-node-112",
        "az": "us-east-1b"
      }
      // ... 12 more entries mapping each shard to its physical location ...
    ]
  }
}
```

#### **b) Bucket Metadata Model (Stored in the Global Bucket Service)**

This model is much simpler and smaller. It's stored in a globally replicated, strongly consistent database (like FoundationDB or a multi-region Spanner/CockroachDB instance) to ensure bucket names are unique and routing information is authoritative.

- **Key:** `bucket_name`

- **Value (Example JSON representation):**

```json
{
  "owner_id": "acct-123456789",
  "primary_region": "us-east-1", // CRITICAL for routing requests from the Edge
  "creation_date": "2022-01-15T11:00:00Z",
  "versioning_enabled": true,
  "replication_rules": [{ "destination_region": "eu-west-1", "async": true }]
}
```

### **5. API Worker Transactionality: Guaranteeing Atomic Writes**

A core challenge is ensuring that a `PUT` operation—which involves multiple steps across different services—is **atomic**. It must either complete fully or fail completely, leaving no partial data. A traditional, locking Two-Phase Commit (2PC) is too slow for this scale.

We achieve this with a **Lightweight, Non-Blocking Transactional Flow** orchestrated by the API Gateway.

Here is the step-by-step process for a `PUT` request:

**Step 1: The Client Request & Idempotency**

The client initiates a `PUT` request. To prevent duplicate uploads on network retries, the client SDK generates a unique **Idempotency Token** for this specific upload attempt. This token is sent with the request.

**Step 2: The PREPARE Phase (The Intention)**

This is the first interaction with the stateful Metadata Service.

- **Action:** The API Gateway sends a `PREPARE_WRITE` command to the Metadata Service. The command includes the object key, the idempotency token, and all object metadata (size, owner, etc.).
- **System Behavior:** The Metadata Service's Raft leader for that partition checks if it has seen this idempotency token before.
  - If **NO**, it atomically writes a _provisional record_ to its replicated log with `transaction_state: "PREPARING"`. This record is invisible to user `GET` requests.
  - If **YES**, it means this is a retry. It looks up the existing record and returns its current status, short-circuiting the operation.
- **Failure Analysis:** If the API Gateway crashes _after_ this step, we are left with a harmless, invisible record in the `PREPARING` state. No data corruption has occurred.

**Step 3: The WRITE DATA Phase (The Heavy Lifting)**

This is the longest, non-blocking phase.

- **Action:** The API Gateway performs erasure coding and streams the data shards in parallel to the designated Storage Nodes.
- **System Behavior:** The Storage Nodes write the data to their local disks. No locks are held on the metadata system during this time, allowing other requests to be processed.
- **Failure Analysis:** If the API Gateway crashes during this phase, we are left with the same invisible `PREPARING` record and some orphaned data chunks on disk. The system is still in a clean, consistent state from a user's perspective.

**Step 4: The COMMIT Phase (The Point of No Return)**

This is the final, atomic step that makes the object visible.

- **Action:** Once a quorum of shards are successfully written, the API Gateway sends a final `COMMIT_WRITE` command to the Metadata Service, referencing the object key and idempotency token.
- **System Behavior:** The Metadata Service Raft leader performs one last, fast, atomic write: it finds the provisional record and flips its `transaction_state` from `PREPARING` to `COMMITTED`.
- **The Guarantee:** The moment this write is committed to the Raft log, the object becomes instantly and consistently visible to all subsequent `GET` requests within the region.
- **Failure Analysis:** If the API Gateway crashes _after_ sending the commit but _before_ replying to the client, the operation has still succeeded. The data is safe. A client retry will be handled by the idempotency check in Step 2.

**Step 5: The Cleanup (The Janitor Process)**

To handle failures in Steps 2 and 3, a background **Janitor Process** runs periodically.

- **Action:** It scans the Metadata Service for records that have been in the `PREPARING` state for longer than a predefined timeout (e.g., one hour).
- **System Behavior:** For each stale record it finds, it initiates a rollback: it deletes the provisional metadata record and sends commands to the Storage Nodes to delete the corresponding orphaned data shards. This ensures that abandoned transaction artifacts are garbage collected, keeping the system clean.

This lightweight flow provides the atomicity of a traditional transaction without the performance bottlenecks, making it perfectly suited for a high-throughput, distributed storage system.

### **6. Failure Scenarios and System Responses**

A robust distributed system is defined by how it handles failures. Here are common failure modes and how our design handles them, from individual components to catastrophic events.
Scenario 1: A Single Storage Node Fails

    Detection: The failed node stops sending heartbeats to our central monitoring/coordination service. It is marked as DOWN.

    Impact on Writes: During a PUT, if the API Gateway tries to write a shard to this failed node, the connection will time out. The gateway will immediately ask the Metadata Service for an alternate node, write the shard there, and proceed. The write operation succeeds with a slight increase in latency.

    Impact on Reads: During a GET, if the API Gateway needs a shard from the failed node, it will fail to retrieve it. It will simply proceed to fetch one of the other redundant shards from a healthy node and reconstruct the object. The read operation succeeds, again with a slight increase in latency.

    Recovery (Self-Healing): The background Healer Process detects that the shards on the failed node are now "under-replicated." For each affected object, it reads the remaining healthy shards, reconstructs the missing ones, and writes them to new, healthy Storage Nodes in the cluster. After a short while, the system returns to a fully redundant state automatically.

Scenario 2: A Metadata Service Node Fails (Follower)

    Detection: The Raft leader of that Partition Group stops receiving heartbeats from the follower.

    Impact: Minimal to none. The Partition Group continues to operate in a degraded state (e.g., 2 out of 3 nodes). Write operations will still succeed as a quorum (2/3) is still possible. Read operations are unaffected.

    Recovery: The cluster's orchestration layer (e.g., Kubernetes) will detect the failed process and automatically attempt to restart it or provision a new one. Once the node rejoins the cluster, the Raft leader will stream all the log entries it missed, bringing it fully up-to-date.

Scenario 3: A Metadata Service Node Fails (Leader)

    Detection: The followers in the Partition Group stop receiving heartbeats from the leader.

    Impact: A brief interruption of service for that specific partition only. The system experiences a "brownout," not a blackout.

        The followers initiate a new leader election via the Raft protocol. This typically completes in a few seconds.

        During this election window, any write requests to this partition will fail or be paused. API Gateways should be configured to retry with exponential backoff.

        Once a new leader is elected, it begins serving traffic.

    Recovery: The new leader takes over seamlessly. The old leader, if it comes back online, will detect a new leader with a higher "term" (Raft's internal clock) and automatically become a follower. The system heals itself.

Scenario 4: An API Gateway Fails Mid-Transaction

This is the transactional failure we discussed.

    Detection: The client's connection is dropped, or our Load Balancer detects the node is unhealthy and removes it from the pool.

    Scenario A: Failure before COMMIT: The metadata record is left in the PREPARING state.

        Impact: The object is invisible to users. The client's write has failed.

        Recovery: The Janitor Process finds this stale PREPARING record after a timeout and cleans up both the metadata entry and the orphaned data shards. The system is left in a consistent state. The client must retry the upload (with the same idempotency token).

    Scenario B: Failure after COMMIT: The write was successful, but the 200 OK might not have reached the client.

        Impact: The object is safely stored and visible.

        Recovery: If the client retries the upload with the same idempotency token, the API Gateway will see that the token has already been used and the object is COMMITTED, and it can immediately return a success message without re-uploading any data.

Scenario 5: An Entire Availability Zone (AZ) Fails

This is a major but expected failure mode for a cloud service.

    Detection: Monitoring systems detect a massive, correlated failure of all nodes within a single AZ.

    Impact:

        Data Durability: No data is lost. Our 10+4 erasure coding scheme is designed to survive the loss of up to 4 shards. The loss of an entire AZ will only eliminate a fraction of the 14 shards for any given object, well within our tolerance.

        Data Availability: The service remains online. GET and PUT requests will continue to succeed. There will be a temporary increase in latency and resource usage as traffic is redirected to the surviving two AZs.

        Metadata Availability: The Metadata Service also remains online. For each Raft-based Partition Group, the two surviving nodes (in the other AZs) will form a quorum and, if necessary, elect a new leader.

    Recovery: This is a high-alert operational event. While the system runs, it is in a degraded state with reduced redundancy. The Healer process will begin working at maximum capacity to regenerate all the lost shards and place them on new nodes within the two surviving AZs. The operations team will work to bring a new AZ online or add capacity to the existing ones.

Scenario 6: A Network Partition between AZs

This is a subtle and dangerous failure. The nodes are up, but they can't talk to each other.

    Detection: Raft groups will fail to achieve a quorum. For example, AZ-A can talk to the outside world but not AZ-B or AZ-C.

    Impact: The Raft protocol's requirement for a majority (quorum) is what saves us from data corruption (a "split-brain" scenario).

        The side of the partition with a majority of nodes (e.g., AZ-B and AZ-C can still talk to each other) will elect a leader and continue to serve both reads and writes.

        The side with the minority (e.g., AZ-A by itself) cannot achieve a quorum. It will be unable to elect a leader and will therefore stop serving write requests for its partitions. It may serve stale reads if configured to do so.

    Recovery: Once the network partition is healed, the isolated nodes in AZ-A will rejoin their Raft groups, see that they are behind, and receive all the updates they missed from the current leader. The system converges back to a fully consistent state.

#### **7. Walkthrough of a Global Write Operation**

Let's trace a user in Tokyo uploading a file to a bucket located in the `us-west-2` (Oregon) region.

1.  **DNS & Connection:** The user's request to `s3.my-cloud.com` is routed via **Anycast** to the **Tokyo Edge Gateway**. TLS is terminated here.
2.  **Routing:** The Tokyo gateway authenticates the user, then asks the Global Bucket Service where `my-bucket` lives. The service replies: `us-west-2`.
3.  **Proxying:** The Tokyo gateway acts as a proxy, forwarding the `PUT` request over the private backbone to the regional **API Gateway in `us-west-2`**.
4.  **Erasure Coding:** The `us-west-2` gateway chunks the file and computes `10+4` erasure code shards.
5.  **Metadata Prepare:** It contacts the `us-west-2` **Metadata Service**, hashing the object key to find the correct Partition Group. It sends a `PREPARE` command to the Raft leader of that group.
6.  **Data Write:** The gateway streams the 14 shards in parallel to 14 different Storage Nodes spread across the three AZs in `us-west-2`.
7.  **Metadata Commit:** Once a quorum of shards are written, the gateway sends a `COMMIT` command to the same Metadata Service Raft leader, which atomically flips the object's state to `COMMITTED`.
8.  **Success ACK:** The success status flows all the way back: from the `us-west-2` gateway, over the backbone to the Tokyo gateway, and finally back to the user's client in Tokyo.

This architecture creates a system that is globally accessible and highly performant at the edge, while being extremely durable, consistent, and scalable at its regional core. It effectively balances the trade-offs between global consistency, regional autonomy, performance, and cost.
