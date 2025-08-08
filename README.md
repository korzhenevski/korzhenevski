# Principal R&D Engineer & System Architect 
*Latency is a tax on reality. Let’s eliminate it.*

I run two parallel tracks:
I lead my own long-horizon R&D projects and consult for a **small number of top-tier teams** where architecture is a true competitive advantage.

The consulting work funds my deep research into infrastructure, trading systems, and performance engineering.

In trading, I have full skin in the game: I risk my own capital with no outside investors.

> If you're building an infrastructure-level system—one that must survive failure, evolve over years, and perform at the limits of hardware—and you need someone who can find a competitive edge in it, let's talk.

📧 yura.nevsky@gmail.com  
🔗 [LinkedIn](https://linkedin.com/in/korzhenevski)

---

## Key Projects

### **[Recommendation Engine](https://github.com/korzhenevski/korzhenevski/blob/master/RecEngine.md) for TikTok-like social app (2023)**
Built a recommendation engine to serve 10K RPS with p99 < 20ms on bare metal. The C++/Go solution used SIMD HNSW + Roaring Bitmaps. I was an early adopter of AWS Graviton (ARM instances), achieving 2x cost efficiency before it became the norm. The system now sustains **p99 < 15 ms**.


### **Blockchain Validator Node for leading Altcoin Mining Pool (2022)**
Accelerated a black-box PoW node that had no docs and limited source code. I reverse-engineered its gossip layer and wrote a custom booster in Go/C++. Optimizing with PGO + AVX2 resulted in a **+16% validator reward** with zero protocol changes.


### **HFT Market Data Lake for a 20+ Quant Trading Firm (2019)**
Engineered a system to query a **50 TB / 500B**-row market data lake in real time, cutting latency from 60+ seconds to milliseconds. I designed the ClickHouse architecture (schema, clustering, OS tuning) long before it became the industry standard. The result: **<50 ms query latency—a 1200x speedup.**


### **[Distributed Financial DB](https://github.com/korzhenevski/korzhenevski/blob/master/AsgardDB.md) for a NASDAQ-listed Fintech (2017)**
Co-designed (with one other engineer) a geo-distributed financial processor that could survive a full datacenter failure. Built on ePaxos and RAMCloud principles, the **system passed Jepsen** — the world's toughest distributed systems test.


### **Security Scanner for Yandex's Global Infrastructure (2015)**
Built a scanner in 4 months to monitor 50K+ servers 24/7, even during DC failures. Deeply integrated with internal infra across 3 datacenters, it saved the company over **$1M vs. a commercial solution**. It's still in use today.

---

## My Engineering Path: 17 Years of Betting on Technology, Not Hype

### 2024–Now — Quant Trading R&D at my lab, 1ms.tech

At my R&D lab, 1ms.tech, I treat trading as a science, not speculation. The process is about forming a hypothesis about a market inefficiency, testing it rigorously, and only accepting discoveries backed by undeniable evidence. It's the antithesis of overfitting models to random noise.

This is the **ultimate application** of my 15 years in systems engineering: building the architecture to turn market chaos into a laboratory for verifiable discoveries.

### 2022–2023 — Mastering the Metal
Switched to a C dialect with near 1:1 instruction mapping, outperforming idiomatic C++ by 60–80% on critical paths.

### 2018–2021 — Full-System Depth
Deepened my expertise in C++, modern Bazel, and binary-level tuning. I worked across the stack—from iOS (Swift, Obj-C) to Flutter (Dart)—to build systems, not just services.

### 2015–2018 — An Early Bet on Cloud-Native
Deployed Kubernetes across multiple datacenters before it was popular. Built high-concurrency systems with async Python long before it was mainstream.

### 2014 — The Go Bet & The Shift to R&D Consulting
This year, I shifted from full-time employment to a model of an **independent R&D expert** for companies. At the same time, I made an early bet on Go, using it to build high-load infrastructure. This dual move defined my career for the years to come.

### 2012–2013 — Pioneering the Modern Web Stack
Launched production SPAs with Angular.js and MongoDB in the pre-MEAN era, making a strategic bet on JSON-native infrastructure while others clung to the LAMP stack.

### 2011–2012 — High-Concurrency Groundwork
Built real-time platforms in Erlang and Ruby. Championed dev/prod parity with Vagrant—four years before Docker existed.

### 2008–2010 — APIs, CDNs & Schema Dynamics
Developed a dynamic-schema API (a precursor to GraphQL) for a VoD platform and built a 10 Gbps video CDN, which was extreme scale for the time.
