# Principal R&D & System Architect 
*Latency is a tax on reality. Let’s eliminate it.*

I partner with a handful of elite teams on their hardest architectural challenges.

🔗 [LinkedIn](https://linkedin.com/in/korzhenevski)

---

## Declassified Engagements

My current R&D is my partners’ unfair advantage. It is never public.
What I can share are the results of past partnerships, declassified after a 1-2 year confidentiality and competitive embargo.

### **[Petabyte-Scale Solana Archival Layer](https://github.com/korzhenevski/korzhenevski/blob/master/SolanaArchive.md)** (2025)

Architected a storage solution to handle hundreds of TBs of high-frequency ledger history. Bypassed standard indexing bottlenecks by designing a custom RocksDB schema with Big-Endian binary keys for locality-aware SST lookups. The C++20 ingestion pipeline utilizes zero-copy Kafka/Protobuf and PGO, successfully optimizing the critical trade-off between ZSTD compression density and read-time CPU latency.

### **[Recommendation Engine](https://github.com/korzhenevski/korzhenevski/blob/master/RecEngine.md) for TikTok-like social app (2023)**
Built a recommendation engine to serve 10K RPS with p99 < 20ms on bare metal. The C++/Go solution used SIMD HNSW + Roaring Bitmaps. I was an early adopter of AWS Graviton (ARM instances), achieving 2x cost efficiency before it became the norm. The system now sustains **p99 < 15 ms**.

### **Blockchain Validator Node for leading Altcoin Mining Pool (2022)**
Accelerated a black-box PoW node that had no docs and limited source code. I reverse-engineered its gossip layer and wrote a custom booster in Go/C++. Optimizing with PGO + AVX2 resulted in a **+16% validator reward** with zero protocol changes.

### Corp Security Architecture for a $10M+ HFT/Crypto Firm (2020-2022)
Designed end-to-end security: Zero Trust + Intel SGX + anti-DDoS + cold wallet protection + secure SDLC. **Result:** 100% compliance (Estonian regulations), zero incidents, full protection of capital and proprietary code.

### Infrastructure for MEV (2020)
Built **200+ custom nodes** processing billions of mempool events in real-time. Developed **topology reconnaissance** and p2p protocol hacks for transaction sniping, generating **$4M+ MEV** revenue.

### **HFT Market Data Lake for a 20+ Quant Trading Firm (2019)**
Engineered a system to query a **50 TB / 500B**-row market data lake in real time, cutting latency from 60+ seconds to milliseconds. I designed the ClickHouse architecture (schema, clustering, OS tuning) long before it became the industry standard. The result: **<50 ms query latency: a 1200x speedup.**

### **[Distributed Financial DB](https://github.com/korzhenevski/korzhenevski/blob/master/AsgardDB.md) for a NASDAQ-listed Fintech (2017)**
Co-designed (with one other engineer) a geo-distributed financial processor that could survive a full datacenter failure. Built on ePaxos and RAMCloud principles, the **system passed Jepsen**: the world's toughest distributed systems test.


### **Security Scanner for Yandex's Global Infrastructure (2015)**
Built a scanner in 4 months to monitor 50K+ servers 24/7, even during DC failures. Deeply integrated with internal infra across 3 datacenters, it saved the company over **$1M vs. a commercial solution**. It's still in use today.

---

## My Engineering Path: 17 Years of Betting on Technology, Not Hype

### 2024–Now — Quant Trading R&D at my lab, 1ms.tech

At my R&D lab, 1ms.tech, I treat trading as a science, not speculation. The process is about forming a hypothesis about a market inefficiency, testing it rigorously, and only accepting discoveries backed by undeniable evidence. It's the antithesis of overfitting models to random noise.

This is the **ultimate application** of my 15 years in systems engineering: building the architecture to turn market chaos into a laboratory for verifiable discoveries.

### 2022–2023 — Mastering the Metal
Adopted a bare-metal C approach in HPC: writing code as a direct conversation with the CPU (MESI, caches, NUMA, prefetching) to turn hardware behavior into predictable performance.

### 2018–2021 — Full-System Depth
Deepened my expertise in C++, modern Bazel, and binary-level tuning. I worked across the stack: from iOS (Swift, Obj-C) to Flutter (Dart) to build systems, not just services.

### 2015–2018 — An Early Bet on Cloud-Native
Deployed Kubernetes across multiple datacenters before it was popular. Built high-concurrency systems with async Python long before it was mainstream.

### 2014 — The Go Bet & The Shift to Independent R&D
Shifted from full-time employment to independent consulting, making an early, successful bet on Go for high-load infrastructure.

### 2012–2013 — Pioneering the Modern Web Stack
Launched production SPAs with Angular.js and MongoDB in the pre-MEAN era, making a strategic bet on JSON-native infrastructure while others clung to the LAMP stack.

### 2011–2012 — High-Concurrency Groundwork
Built real-time platforms in Erlang and Ruby. Dev/prod parity with Vagrant: four years before Docker existed.

### 2008–2010 — APIs, CDNs & Schema Dynamics
Worked with a talented team of engineers to develop a dynamic-schema API (conceptually similar to what later became GraphQL) for a VoD platform, and to build a 10 Gbps video CDN: extreme scale for the time.

### 2007 — Early Start
At 14, I left formal schooling to work full-time as a software engineer: thanks to an exceptional team who trusted me, supported my growth, and gave me the opportunity to start my career years ahead of schedule.
