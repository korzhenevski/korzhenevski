## Plutos/AsgardDB: A Decentralized Architecture for High-Throughput, Fault-Tolerant Financial Transactions (2018)

*How do you process financial transactions with absolute reliability, even as your data centers fail in the most creative ways, all without atomic clocks and on commodity hardware? This was the challenge we faced. The answer lay in a radical simplification of distributed consensus, inspired by the theoretical elegance of ePaxos and the raw, in-memory speed of RAMCloud.*

### Abstract

This paper details the architecture and engineering principles behind Plutos, a distributed system for high-throughput, reliable financial transaction processing. We diverge from traditional monolithic database architectures and complex consensus protocols, instead proposing a three-tier system built on stateless, in-memory processing nodes. The system's core innovation, born from our R&D on its industrial predecessor AsgardDB, is a coordinator-less, self-healing routing mechanism that obviates the need for external services like Zookeeper or etcd, dramatically simplifying deployment and enhancing resilience. A key insight from 2017 that shaped AsgardDB was the decision to eliminate data mutability at the logical level, which allowed us to bypass many of the traditional bottlenecks in distributed database scalability. We will dissect both the open-source Plutos architecture and the underlying principles of the AsgardDB storage engine, explaining the design trade-offs and engineering "wow moments" that enable its performance and fault tolerance.

---

### 1. Introduction: The Tyranny of the Monolith and the Promise of Decentralization

In the world of large-scale financial services—handling billions of transactions annually for tens of millions of users—the traditional Oracle-style database monolith is both a fortress and a prison. It offers strong consistency guarantees but at the cost of crippling licensing fees, vertical scaling limits, and a terrifyingly large single point of failure. A master database failure isn't a hiccup; it's a multi-minute (or hour-long) outage, an eternity in a world where uptime is measured in nines.

The rise of public blockchains presented a new paradigm, but their direct application in a trusted corporate environment is misguided. The computational waste of Proof-of-Work and the high latency of public consensus are solutions to a problem we don't have: lack of trust between participants.

Our R&D began with a simple question: What if we could build a system with the consistency of a monolith and the scalability of a decentralized network, but tailored for a high-trust, low-latency environment? The answer was rooted in a key insight from our work on AsgardDB in 2017: **by treating the transaction log as an immutable, append-only data structure for each account, we could eliminate the most complex aspects of distributed consensus—namely, distributed locking and atomic multi-key updates.**

This led us to a design that borrows heavily from two seminal academic projects:
1.  **Egalitarian Paxos (ePaxos):** The idea of a leaderless protocol that optimizes for low latency in conflict-free scenarios. We embraced its philosophy of allowing independent operations to proceed in parallel without a central bottleneck.
2.  **RAMCloud:** The principle of keeping the "hot" working set of data entirely in memory for microsecond-level access, while ensuring durability through a high-speed, persistent log. This directly inspired our stateless processing nodes.

This paper is the story of that design, from its architectural principles to the fine-grained engineering decisions that make it work.

---

### 2. Architectural Overview: A Symphony of Specialized Tiers

The system is composed of three horizontally scalable, stateless tiers, a design that enforces a strong separation of concerns.

**Diagram 1: The Three-Tier Architecture of Plutos**

*   **Tier 1: PlutoAPI (Stateless API Gateway)**
    The outermost layer. Its role is purely mechanical: terminate client HTTP+JSON connections, translate requests into an internal, high-performance binary RPC format, and forward them to the appropriate processing node based on a local routing cache. It holds no business logic.

*   **Tier 2: Plutos (Stateless In-Memory Processing Nodes)**
    The computational core of the system. This is a cluster of identical, stateless nodes responsible for all transaction validation and processing.
    *   **Stateless by Design:** Nodes maintain no persistent state on disk. Their memory acts as a high-speed cache for the "hot" portion of the global state (i.e., the tail of active account transaction chains). A node can be terminated and restarted instantly, becoming immediately available to the cluster.
    *   **Per-Account Sharding:** The entire `uint64` account space is partitioned across the Plutos cluster. At any given moment, a single node is exclusively responsible for a specific account. This is the cornerstone of our consistency model, eliminating the possibility of race conditions and double-spends at the application layer.
    *   **Sequential Per-Account Processing:** Within a single node, all operations for a given account are serialized through a simple lock, ensuring that its transaction chain is built with perfect, linear consistency.

*   **Tier 3: PlutoDB (The Durability Layer)**
    The system's source of truth. Its sole responsibility is to atomically persist and retrieve transaction and account setting records. In the open-source Plutos, this is a simple `sqldb` adapter for MySQL. In the production system, this is **AsgardDB**, a purpose-built distributed storage engine.

---

### 3. Core Algorithms & Engineering Primitives

#### 3.1. The Per-Account Ledger Model
Instead of a global balance table, each account is modeled as an independent, cryptographically-linked chain of transactions.

*   **Structure:** Every transaction contains a `prev_hash` field, which is the SHA-256 hash of the preceding transaction from the *same sender*. The first transaction for any account has a zero `prev_hash`.
*   **Inherent Double-Spend Prevention:** A `prev_hash` can only be consumed once. Any attempt to submit a second transaction with the same `prev_hash` is a protocol violation and is rejected. This provides the same security as a UTXO model but is simpler to reason about for account-based systems.
*   **Natural Idempotency:** This structure makes transfer requests naturally idempotent. A client retrying a request due to a network timeout will simply resubmit the same transaction, which, if already processed, will be recognized and rejected as a double-spend, returning the original successful result.

#### 3.2. The "Wow Moment": Coordinator-less, Self-Healing Routing
This is the system's most elegant and unconventional component. We completely eliminate the need for external coordination services like Zookeeper or etcd for managing routing tables. Instead, we **weaponize routing errors for state propagation.**

The algorithm is a form of reactive, gossip-based discovery:

1.  A `PlutoAPI` gateway holds a (potentially stale) local cache of the routing table (e.g., `Account A -> Plutos-Node-1`). It optimistically sends a request for Account A to `Plutos-Node-1`.
2.  `Plutos-Node-1` receives the request and performs an ownership check on Account A.
3.  **The Stale Route Scenario:** The cluster has reconfigured, and Account A is now owned by `Plutos-Node-2`. `Plutos-Node-1` detects it is no longer the owner.
4.  **The Self-Healing Response:** Instead of forwarding the request, `Plutos-Node-1` **rejects it with a specific routing error.** Crucially, it **attaches its current, up-to-date routing table to the error response.**
5.  The `PlutoAPI` gateway receives this error, atomically updates its local routing cache with the new information, and immediately retries the original request, this time sending it to the correct node, `Plutos-Node-2`.

This mechanism is beautiful in its simplicity and robustness.
*   **No Central Point of Failure:** The system's liveness does not depend on an external coordinator.
*   **Extreme Simplicity:** Nodes do not need to run complex consensus protocols to agree on routing changes.
*   **Resilience:** Temporary inconsistencies in routing tables across the cluster are harmless. They resolve themselves organically as traffic flows, causing at worst a few extra round-trips for a given request.

#### 3.3. The Transaction Flow: `Preloader` and `Pusher`
The processing path within a Plutos node is heavily optimized for low latency.

```go
// Simplified Go code for the core processing path
func (p *Processor) ProcessTransfer(ctx context.Context, t pt.Transfer) (pt.TransferResult, error) {
    // 1. Serialize access to a single account
    p.mu.Lock()
    defer p.mu.Unlock()

    // 2. Lazy-load account state from durable storage if not in cache
    // The preloader uses singleflight to prevent thundering herds.
    if err := p.preloader.Preload(ctx, t.Sender); err != nil {
        return res, errors.Wrap(err, "account preloading")
    }

    // 3. Validate the request against the cached state (prev_hash, balance, signature)
    // ... validation logic ...

    // 4. Generate new transaction objects in memory
    newTxns := createTxnsFrom(t)

    // 5. Push to the durability layer and notify recipient nodes
    if err := p.pusher.Push(ctx, newTxns); err != nil {
        // On push failure, invalidate the cache. The next attempt will fetch the
        // ground truth from the durable store.
        p.preloader.Reset(ctx, t.Sender)
        return res, errors.Wrap(err, "push failed")
    }

    // 6. Commit to the in-memory cache
    p.chain.PutTo(t.Sender, newTxns)
    return buildResult(newTxns), nil
}
```

- **The `Preloader`:** To mitigate the "thundering herd" problem on cold accounts, the `Preloader` uses a `singleflight` group. This ensures that for any given account, only one concurrent request will trigger a fetch from `PlutoDB`; all others will wait for and share the result.
- **The `Pusher` Interface:** The logic for persisting transactions to `PlutoDB` and notifying recipient nodes is abstracted away behind a `Pusher` interface. This decouples the core processing logic from the persistence implementation, allowing for different backends (e.g., `sqldb`, `AsgardDB`, even a Kafka topic for analytics) to be plugged in seamlessly.

---

### 4. AsgardDB: The Production Engine vs. The Open-Source Stand-in

The open-source `sqldb` is a deliberate simplification, designed for accessibility. The real power lies in **AsgardDB**, our purpose-built, distributed, fault-tolerant storage engine.

*   **Architecture:** AsgardDB follows a coordinator-replica model. Coordinators are stateless and handle the commit protocol. Replicas store data and are distributed across availability zones for high availability.
*   **The Commit Protocol (A Non-Blocking 2PC Variant):**
    1.  **Prepare Phase:** A coordinator proposes a write for a set of keys with a sequence number (`seq_num`). Replicas check if this `seq_num` is higher than any previously committed or prepared `seq_num` for those keys. If not, they reject the proposal and suggest a higher `seq_num`. The key is that replicas *remember* the highest prepared `seq_num`, effectively creating a soft lock without blocking other reads.
    2.  **Commit Phase:** Once the coordinator achieves consensus on a `seq_num` across a quorum of replicas, it issues a commit command. Replicas then atomically apply the write. This design is highly optimized for the common case of no concurrent writes to the same key.
*   **Time and Consistency (A Pragmatic TrueTime):** AsgardDB does not require atomic clocks. Instead, it operates under the assumption of a bounded clock skew (`ε`) across the cluster, a guarantee easily provided in a managed data center environment. When a potential write-write conflict is detected during the Prepare phase, the coordinator simply waits for the duration of `ε` before retrying. This brief pause ensures that any in-flight operation from another node would have arrived, thus allowing for a correct ordering of events. It's a software-based solution that provides the benefits of time-based ordering without the cost and complexity of specialized hardware.

---

### 5. R&D Methodology & Verification

In finance, trust is non-negotiable. Our R&D process was built on a foundation of rigorous, adversarial testing.

*   **Jepsen Testing:** From its inception, AsgardDB was validated using the Jepsen framework. We developed a custom "nemesis" to simulate network partitions, node failures, process pauses, and severe clock skew. Passing Jepsen was the non-negotiable gate for considering the system production-ready. It provided mathematical confidence in the system's ability to maintain data integrity under catastrophic conditions.
*   **Load Testing (`bench`):** A custom load generation tool, `bench`, was created to measure throughput, latency percentiles, and scalability under various workloads, ensuring that performance goals were met.

---

### 6. Engineering Insights & Lessons Learned

1.  **Complexity is a Liability, Not a Feature.** Every external dependency, every complex protocol is a hidden operational cost and a potential point of failure. The single most impactful decision was to eliminate the external coordinator.
2.  **Shift Complexity from the Synchronous Path to the Asynchronous Path.** The critical path for a transaction is extremely fast because complex tasks like state synchronization and routing updates are handled reactively and asynchronously.
3.  **Leverage Your Trust Model.** By designing for a trusted corporate environment, we could discard the computationally expensive machinery of public blockchains and focus on pure performance and fault tolerance.

---

### 7. Future Work

*   **Dynamic Shard Rebalancing:** Implementing an automated mechanism for Plutos nodes to dynamically split or merge account ranges based on load, mitigating "hot shard" problems.
*   **Sidechain Anchoring:** Developing a formal process to periodically anchor the state hash of the Plutos system into a higher-order, legally significant ledger (e.g., a central bank's masterchain).

---

### 8. Conclusion

The Plutos/AsgardDB architecture is a testament to the power of first-principles thinking in system design. By challenging the conventional wisdom that financial systems must be monolithic or rely on complex consensus, we have created a model that is simultaneously simpler, more resilient, and highly scalable. It demonstrates that the most elegant solutions are often found not by adding more layers of complexity, but by strategically removing them.

### Authorship and Acknowledgements

This work is the result of a multi-year research and development effort led by **[Yuri Korzhenevsky](https://github.com/korzhenevski)** and **[Nikifor Seryakov](https://github.com/nikandfor)** at the Yuri Korzhenevsky R&D Center.

We wish to extend our heartfelt gratitude to the many talented individuals who contributed to this project through their insightful advice, rigorous critiques, and unwavering support. Their feedback was instrumental in shaping the final architecture and pointing out flaws in our early assumptions. We would especially like to thank **Maxim Strakhov (Apple Inc.)**, **Alexander Guzev (Fantom)**, and **Danila Rukhovich (European University Researcher)** for their invaluable contributions during the formative stages of this research.

An early preprint version of the concepts behind this system, titled "AsgardDB: Fast and Scalable Financial Database," was published on ResearchGate and can be accessed via the following link:

**[AsgardDB: Fast and Scalable Financial Database on ResearchGate](https://www.researchgate.net/publication/326816360_AsgardDB_Fast_and_Scalable_Financial_Database)**
DOI: 10.13140/RG.2.2.26744.34565

> © 2025 Iurii Korzhenevskii. All Rights Reserved. The system architecture and "Deep Dive" methodologies presented here are proprietary R&D.
> 
> You may share this document unchanged with attribution.
> 
> Copying the architectural patterns for commercial products or claiming authorship is prohibited.
