# Solana Archival Research: High-Performance Storage Architecture

**Status:** Draft / Research

**Context:** Designing a scalable history access layer for Solana (Genesis to Head).

## 1. The Foundation: Technology & Constraints

Solana’s ledger is not just large; it is hyper-active. With the rise of High-Frequency Trading (HFT), the ledger grows at an accelerated rate, pushing the total dataset size into the **hundreds of Terabytes**. 

### Why RocksDB?
We have selected **RocksDB** as the storage engine.

**Justification:** RocksDB is the industry standard for LSM (Log-Structured Merge-tree) storage on SSDs. It allows us to manage the massive write throughput required by Solana's block times while providing a mechanism (compaction) to organize historical data efficiently.

### The Storage vs. Compute Trade-off
To handle hundreds of TBs, raw storage is not enough; efficient compression is mandatory.

**Compression:** **ZSTD** is the candidate of choice.

**The Challenge:** High compression ratios save disk space but increase CPU load during reads.

**Action Item:** We must benchmark the specific trade-off between ZSTD compression levels. The goal is to maximize storage density without creating a CPU bottleneck during high-QPS read scenarios.

---

## 2. Data Architecture: Optimized for Physical Locality

The performance of an archive is defined by how the data is laid out on the disk. Random I/O is the enemy.

### Key Format: The Binary Standard

**Requirement:** Keys must be stored in strictly **binary format**. String keys are prohibited to minimize overhead.

**Big-Endian Encoding:** All numeric components (Slots, Transaction Indexes) must use Big-Endian (BE).

In RocksDB, keys are sorted lexicographically by byte. Big-Endian ensures that the *byte order* matches the *numerical order*. This is the mathematical foundation required for efficient range scans (e.g., "Get all transactions in Slot X").

### Schema & Column Families
Data will be segregated into **Column Families (CF)**. This allows us to tune caching, block sizes, and compaction strategies independently for different data types.

| Column Family | Key Structure (Bytes) | Value Structure | Optimization Strategy |
| :--- | :--- | :--- | :--- |
| **BlockMeta** | `[8: Slot_BE]` | Protobuf: `{ block_hash, parent, ... }` | Standard point lookups. |
| **TxBySignature** | `[64: Signature]` | Protobuf: `{ slot, index, data, ... }` | **Heavy CF.** Focus on high compression ratio. |
| **TxByAddress** | `[32: Address]` + `[8: Slot_BE]` + `[2: Index_BE]` | `[64: Signature]` (Pointer) | **Index CF.** Use *Prefix Bloom Filters* to skip SSTs. |
| **TxSigsBySlot** | `[8: Slot_BE]` + `[2: Index_BE]` | `[64: Signature]` (Pointer) | **Index CF.** Optimize for range scans. |

*   **Values:** We use **Protobuf 3**. It offers forward-compatibility for schema evolution and is significantly faster/smaller than JSON.
*   **PlainTable Format:** For Index CFs (where values are small pointers), we will test RocksDB's `PlainTable` format. If the index fits in RAM, this eliminates block cache overhead entirely.

---

## 3. The Ingestion Pipeline

### A. Mass Historical Import (The "Cold" Path)
Loading history is a different engineering challenge than keeping up with the tip.
*   **Batch Mode:** We leverage `rocksdb::WriteBatch` with the Write-Ahead Log (WAL) **disabled**.
*   **Manual Compaction:** Automatic compaction competes for resources. During bulk loads, we will trigger `CompactRange` manually after large batches.
*   **Data Sources:**
    1.  `ledger/` directory from a synced Agave node.
    2.  CAR-based archives (e.g., Triton’s Old Faithful).
*   **The "Golden Check":** Historical archives (even commercial ones like Triton/Helius) have historically suffered from data corruption.
    *   *Validation:* We will re-compute the **Bank Hash** for a specific historical slot using *our* DB data and compare it against the official snapshot hash. This provides cryptographic proof of data integrity.

### B. Real-Time Ingestion (The "Hot" Path)
*   **Architecture:** A native **C++20/23** daemon.
    *   **Philosophy:** No complex OOP hierarchies. Simple, functional, thread-per-core design.
    *   **Stack:** `librdkafka` (Consumption) → `protobuf` (Parsing) → `rocksdb` (Storage).
*   **Performance:**
    *   **PGO (Profile-Guided Optimization):** Compiling the daemon with PGO can yield a **20-30%** increase in throughput for this specific workload.
    *   **Idempotency:** The Kafka offset is the only cursor. Re-playing a message must be safe.
*   **Analytics (OLAP):** A parallel pipeline can write to **ClickHouse**. ClickHouse’s native batch ingestion aligns perfectly with analytical queries, serving as a complement to the RocksDB operational store.

---

## 4. Access & Scaling Strategy

### The Read Layer
*   **Language:** **Go (Golang)**.
*   **Implementation:** Using CGO to access RocksDB in **Read-Only** mode.
    *   *Hypothesis:* This allows us to combine the raw speed of C++ storage with Go's robust concurrency and networking ecosystem (gRPC/HTTP).

### Infrastructure Topology
How do we host hundreds of TBs?
*   **Strategy:** **Fleet of Medium Instances** (10-15 servers) > **Few Monster Instances** (2-5 servers).
    *   *Reasoning:* "Monster" instances often suffer from noisy neighbor issues on IOPS. Medium instances (bare metal or VM with NVMe passthrough) allow for better isolation and low-level optimizations (like `io_uring` or `mmap`).
    *   *Recovery:* Smaller shards mean faster **Time to Recovery (TTR)** if a node fails.

### Sharding & Aggregation
*   **Sharding Key:** Data will be sharded by **Key Prefix** (e.g., the first 1-2 bytes of the Address/Signature).
*   **The "Scatter-Gather" Pattern:**
    Since data is sharded, the Go RPC layer will act as an aggregator. It performs a wide fan-out query to relevant shards and aggregates the results—similar to how search engines retrieve document hits. 
*   **Network:** Low latency is critical. All nodes must reside in the same Availability Zone with **40Gbps+** interconnects.

---

### Useful References
*   [RocksDB: Creating and Ingesting SST Files](https://github.com/facebook/rocksdb/wiki/creating-and-ingesting-sst-files)
*   [Triton Old Faithful Report (Context on Data Quality)](https://docs.triton.one/project-yellowstone/old-faithful-historical-archive/public-report-epoch-208)
*   [Reference Implementation: Archival RPC](https://github.com/dexterlaboss/archival-rpc)
