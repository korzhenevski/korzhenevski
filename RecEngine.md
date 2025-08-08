## Vector Search at Scale: An HFT Approach to a Mainstream Problem

### Introduction

I was brought in to consult on a recommendation engine. The story was a familiar one in the industry: a promising product built on a high-level stack (Python, managed cloud services) was hitting the dual walls of unpredictable latency and spiraling operational costs. The business logic was sound, but the implementation was bleeding efficiency. The request was to "make it faster." My assessment was that the problem wasn't something to be *fixed*; the entire data-plane needed to be rebuilt from first principles.

This document details the architectural philosophy we applied, borrowing heavily from the world of high-frequency trading (HFT) and low-level systems programming. We treated the hardware as the platform, not an abstraction layer. The result was a C++ core engine where performance is a designed property, not an emergent one.

### 1. The Data Layer: RocksDB as the System of Record

Before any in-memory heroics, you need a persistent foundation. The naive approach is to throw everything into a relational database or a simple document store. We chose **RocksDB**, not as a generic key-value store, but as a tunable engine for our specific workload.

The system ingests a high-velocity stream of user interaction events from Kafka. RocksDB's Log-Structured Merge-Tree (LSM) architecture is fundamentally suited for this high-throughput, write-heavy pattern. But the real value came from its configurability:

*   **Column Family Segregation:** We didn't use a single keyspace. Feature vectors, and media metadata were segregated into different column families. Each family was tuned independently—for instance, event logs used a different block cache policy and filter strategy than the less-frequently-updated feature vector data. This prevents I/O contention between different access patterns.
*   **Terabyte-Scale Design:** The architecture was designed assuming the database would grow into the terabyte range. This meant strict schema control. All string-based ObjectIDs were hashed or mapped to 64-bit integer IDs upon ingestion. Storing variable-length strings as primary keys is an unforgivable performance sin in a system of this scale.
*   **Blacklist Filtering as a First-Pass Sieve:** One of the most critical uses of RocksDB was for pre-filtering. Before any expensive ANN search or model inference, we perform a batch lookup against column families containing user-specific blacklists (e.g., `user_id -> set_of_viewed_media_ids`). A highly-optimized RocksDB `MultiGet` on a bloom-filtered column family can eliminate 90% of irrelevant candidates for a fraction of the cost of running them through the full pipeline. This is the first and cheapest line of defense against wasted computation.

### 2. The Core Engine: A Deep Dive into Approximate Nearest Neighbors (ANN)

The heart of the system is the ANN engine. The market is flooded with libraries, but most are unsuitable for a serious production environment. Our evaluation was brutal and focused on implementation, not marketing.

#### The Landscape: Beyond the Hype

*   **FAISS:** The academic gold standard from Facebook AI. Powerful and feature-rich, but can be a monolithic dependency. Its C-style API and resource management can be cumbersome to integrate cleanly into a modern C++ service.
*   **HNSWlib:** The most popular HNSW (Hierarchical Navigable Small World) implementation. It’s fast, but its performance is highly dependent on compilation flags and the quality of its distance function implementation. It's a component, not a full solution.
*   **The Ghost of ScaNN:** We couldn't use Google's ScaNN directly, but we studied the research papers extensively. The key insight wasn't the graph algorithm itself, but the concept of **anisotropic vector quantization** and the absolute obsession with hardware-aware, SIMD-optimized distance calculations. This became our guiding principle: the graph is just a way to reduce the search space; the real work happens in the dot product.

#### The Real Bottleneck: SIMD-Optimized Distance

The performance of any ANN library is almost entirely dictated by the quality of its low-level SIMD kernel for the chosen distance metric (L2, Inner Product, etc.). This is non-negotiable. A library without a first-class, architecture-specific SIMD implementation is a toy.

This led us to **Google's Highway library**. Highway provides a portable C++ interface for SIMD, allowing us to write a single piece of logic that compiles down to efficient AVX2/AVX-512 instructions on x86 and, crucially, to NEON on ARM64. The ability to target ARM is a strategic advantage, as modern ARM server CPUs (like AWS Graviton) offer compelling performance-per-watt and a cleaner, more modern vector instruction set than the fragmented x86 AVX landscape.

We ultimately selected **`unum-cloud/usearch`** because it embodies this philosophy. It is a lightweight, modern C++ library whose core is a well-written, SIMD-based distance kernel. It was chosen not for its feature list, but for the quality of its most critical loop. This design choice also made it strategically compatible with the vector search indexes being developed for databases like ClickHouse, which were following a similar design path.

### 3. Beyond a Single File: The Anatomy of a Production-Grade C++ Service

A `service.c` file compiled with `gcc` is not a system. The difference between an amateur prototype and a production engine lies in the ecosystem and the discipline it enforces.

*   **Build System (Bazel):** We used Bazel for its hermetic and reproducible builds. In a low-level environment with complex dependencies and architecture-specific flags, the ability to guarantee an identical build artifact every time, regardless of the host machine, is essential.
*   **The "Missing" Standard Library (Abseil):** Standard C++ is insufficient for high-performance services. We relied heavily on Google's Abseil library for production-tested components:
    *   `absl::flat_hash_map` was used everywhere instead of `std::unordered_map` for its superior cache performance.
    *   `absl::Mutex` for robust concurrency control.
    *   A standardized logging and command-line flagging system for service configuration and observability.
*   **Schema-Defined Communication (gRPC):** All service boundaries were defined with Protobuf and implemented with gRPC. This enforces a strict contract and allows for high-performance, low-overhead communication between system components.
*   **Continuous Measurement (Testing & Benchmarking):** Performance is not a one-time achievement. Every critical data structure and algorithm was accompanied by a `google/benchmark` micro-benchmark. This allowed us to track the performance impact of every single change and catch regressions before they hit production. Unit tests (`google/test`) ensured correctness, but benchmarks ensured speed.

### 4. The Art of Latency Control: Optimizing the Hot Path

Once the core components were in place, the focus shifted to optimizing the end-to-end retrieval pipeline.

*   **Candidate Re-ranking with SIMD:** After the ANN search retrieves a small candidate set (e.g., 100-200 items), this set must be sorted based on a secondary ranking score. A naive call to `std::sort` is inefficient for such small arrays due to its branching logic. Instead, we implemented **branchless, data-parallel comparison and swap routines using SIMD intrinsics**. This is a classic HFT technique that transforms the sorting problem into a predictable, fixed-latency sequence of vector operations, eliminating the performance jitter of conditional branches.
*   **Data-Oriented Design:** All hot-path data was structured to maximize cache locality. Instead of a `std::vector` of `struct`s (Array of Structures), we used `struct`s of `std::vector`s (Structure of Arrays). This means all floating-point values for a given feature are contiguous in memory, making them ideal for loading into SIMD registers.

### Conclusion: The HFT Verdict

Bringing a low-latency mindset to a mainstream recommendation problem reveals a simple truth: most performance issues are not algorithmic, they are implementational. The cloud and high-level languages have trained a generation of engineers to ignore the machine.

A high-performance system is built on a foundation of disciplined choices: choosing data structures that respect cache lines, writing code that maps cleanly to the CPU's vector units, and measuring the latency of every component relentlessly. By treating the problem with the rigor of an HFT system—where every nanosecond is scrutinized—we built an engine that was not just faster, but fundamentally more efficient and scalable than its predecessor. Performance, in the end, is not a feature you add. It is a feature you design.
