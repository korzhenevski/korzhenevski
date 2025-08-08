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

## Engineering Journey: 17 Years of Strategic Bets Over Hype

**2024–Now** — *Quant Trading R&D* `1ms.tech`  
> Synthesizing 15+ years of systems experience into market-structure-aware alpha generation.  
> Goal: make chaotic L2/L3 order flow legible, tradable, and architecturally sound.

**2022–2023** — *Bare-Metal Control*  
> Built in a C dialect with near 1:1 instruction mapping.  
> Outperformed idiomatic C++ by **60–80%** on critical paths.

**2018–2021** — *Full-System Depth*  
> Deep **C++**, modern **Bazel**, binary-level tuning.  
> Cross-stack fluency: **iOS** (Swift, Obj-C), **Flutter** (Dart) — to build **systems**, not just services.

**2015–2018** — *Cloud-Native, Early*  
> Kubernetes across multiple datacenters — before it was popular.  
> High-concurrency async Python (Django, Flask) — before `async` was mainstream.

**2014** — *The Go Bet*  
> Adopted **Golang** early. Built high-load infra using core concurrency primitives.

**2012–2013** — *The Modern Web, Before It Was a Thing*  
> Production Angular.js SPAs and MongoDB — pre-MEAN era.  
> Strategic bet on JSON-native infra while others clung to LAMP-stack.

**2011–2012** — *High-Concurrency Groundwork*  
> Real-time platforms in **Erlang** + **Ruby**.  
> Dev/Prod parity with **Vagrant** — 4 years before Docker.

**2008–2010** — *APIs, CDNs & Schema Dynamics*  
> Dynamic-schema API (pre-GraphQL) for VoD platform.  
> Built **10 Gbps video CDN** — extreme scale for the time.

---

