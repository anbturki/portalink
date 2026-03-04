# Language Choice: Rust vs Go vs Others

> Evidence-based comparison for building an ngrok alternative focused on tunneling.

## Sources

- [Rust vs Go 2025 - JetBrains](https://blog.jetbrains.com/rust/2025/06/12/rust-vs-go/)
- [Rust vs Go - Bitfield Consulting](https://bitfieldconsulting.com/posts/rust-vs-go)
- [Rust vs Go Performance Benchmarks 2025](https://markaicode.com/rust-vs-go-performance-benchmarks-microservices-2025/)
- [Go vs Rust vs Python for Infrastructure 2026](https://dasroot.net/posts/2026/02/go-vs-rust-vs-python-infrastructure-software-2026/)
- [Tokio vs Goroutines Under Adversarial Load](https://medium.com/@chopra.kanta.73/tokio-versus-goroutines-latency-under-adversarial-load-f806e6b5f0c1)
- [Memory Consumption of Async Tasks](https://pkolaczk.github.io/memory-consumption-of-async/)
- [rathole benchmarks vs frp](https://github.com/rapiz1/rathole/blob/main/docs/benchmark.md)
- [rathole HN Discussion](https://news.ycombinator.com/item?id=39638068)
- [Tokio - Official Site](https://tokio.rs/)
- [Evolution of Async Rust - JetBrains](https://blog.jetbrains.com/rust/2026/02/17/the-evolution-of-async-rust-from-tokio-to-high-level-applications/)
- [Rust Cross-Compilation & Binary Sizes](https://codepitbull.medium.com/cross-compilation-for-rust-and-how-to-reduce-binary-sizes-by-88-269decea50c1b)
- [Rust vs Go Deployment](https://users.rust-lang.org/t/rust-vs-golang-deployment-statically-linked-binaries/68760)
- [Rust Microservices vs Go - Scand](https://scand.com/company/blog/rust-vs-go/)
- [Concurrency Go vs Rust - Leapcell](https://dev.to/leapcell/concurrency-in-go-vs-rustc-goroutines-vs-coroutines-27f5)
- [awesome-tunneling - GitHub](https://github.com/anderspitman/awesome-tunneling)

---

## 1. Why Only Rust and Go Are Serious Contenders

Based on the [awesome-tunneling](https://github.com/anderspitman/awesome-tunneling) list of 40+ projects:

| Language     | Number of tunnel projects | Notable examples                          | Suitable? |
| ------------ | ------------------------- | ----------------------------------------- | --------- |
| **Go**       | ~25+                      | frp (100k+ stars), chisel, sish, piko     | Yes       |
| **Rust**     | ~7+                       | rathole (10k+ stars), bore, wstunnel      | Yes       |
| C/C++        | ~4                        | ZeroTier, n2n, gsocket                    | Too low-level |
| TypeScript   | ~5                        | Tunnelmole, pipenet, h2tunnel             | Performance ceiling |
| Python       | ~3                        | PageKite, EXPOSE                          | Not suitable |
| Java         | ~2                        | Port Buddy, ngtor                         | Too heavy |
| PHP          | 1                         | expose (beyondcode)                       | Not suitable |

**Go dominates** the tunnel ecosystem by project count. **Rust is the performance-focused alternative** gaining traction.

---

## 2. Performance Comparison

### 2.1 Throughput (rathole vs frp — direct tunnel comparison)

Source: [rathole benchmark docs](https://github.com/rapiz1/rathole/blob/main/docs/benchmark.md)

- rathole (Rust) achieves **higher throughput** than frp (Go) and is **more stable** when handling a large volume of connections.
- Real-world validation from rathole author: **2x throughput improvement** over frp, attributed to "carefully designed protocol, de/serialization, and avoiding buffer copying" — not just the language.
- At QPS from 1 to 3000: **identical latency** between rathole and frp.
- At QPS of 4000: **frp starts reporting errors** and latency grows. rathole remains stable.

### 2.2 General Benchmarks (Rust vs Go)

Source: [Rust vs Go Performance 2025](https://markaicode.com/rust-vs-go-performance-benchmarks-microservices-2025/)

- Actix Web (Rust) vs Go standard HTTP library in high-performance JSON processing:
  - Rust: **1.5x faster throughput**
  - Rust: **20% lower memory usage** under high concurrency (1000 concurrent users)
- Binary-trees benchmark: Rust showed **up to 12x better performance** than Go.

### 2.3 Concurrent Connections & Memory

Source: [Memory Consumption of Async Tasks](https://pkolaczk.github.io/memory-consumption-of-async/)

- Rust tokio: unbeatable memory efficiency with high numbers of concurrent tasks.
- Go goroutines: consumed more than 50% of RAM required by Rust threads at 10k concurrent tasks.
- Go's garbage collector causes **periodic CPU spikes** during collection cycles, especially under heavy load.

### 2.4 Tail Latency Under Stress

Source: [Tokio vs Goroutines Under Adversarial Load](https://medium.com/@chopra.kanta.73/tokio-versus-goroutines-latency-under-adversarial-load-f806e6b5f0c1)

- Rust (tokio): Predictable latency, no GC pauses.
- Go: A Go-based trading system handling 50,000 concurrent connections showed **tail latencies spiking to 200+ ms** during memory pressure, causing goroutine scheduler degradation.

### Summary Table

| Metric                  | Rust                           | Go                              |
| ----------------------- | ------------------------------ | ------------------------------- |
| Throughput              | Higher, more stable under load | Good, degrades at high QPS      |
| Latency (normal)        | Same as Go                     | Same as Rust                    |
| Latency (under stress)  | Predictable, no GC             | GC-induced spikes               |
| Memory usage            | Much lower (~500KB binary)     | Higher (GC runtime overhead)    |
| Concurrent connections  | Handles more before degrading  | Scheduler degrades under memory pressure |
| Binary size             | ~500KB-2.1MB                   | ~2.5MB                          |

---

## 3. Developer Experience Comparison

### 3.1 Learning Curve

Source: [JetBrains Rust vs Go](https://blog.jetbrains.com/rust/2025/06/12/rust-vs-go/)

| Aspect            | Rust                                    | Go                                     |
| ----------------- | --------------------------------------- | -------------------------------------- |
| Difficulty         | Steep (ownership, lifetimes, async)     | Gentle (goroutines are intuitive)      |
| Error handling     | `Result<T, E>`, pattern matching        | `if err != nil` pattern               |
| Concurrency model  | async/await + tokio (stackless coroutines) | goroutines + channels (stackful, built-in) |
| Type system        | Powerful (generics, traits, enums)      | Simpler (interfaces, less ceremony)   |

### 3.2 Async Networking Ecosystem

| Component         | Rust                           | Go                              |
| ----------------- | ------------------------------ | ------------------------------- |
| Async runtime     | tokio (mature, production-proven) | Built-in goroutines + channels |
| HTTP              | hyper, axum, reqwest           | net/http (stdlib)              |
| TLS               | rustls (pure Rust)             | crypto/tls (stdlib)            |
| WebSocket         | tokio-tungstenite              | gorilla/websocket              |
| QUIC              | quinn                          | quic-go                        |
| Multiplexing      | yamux crate, or custom         | yamux, smux, or HTTP/2         |

Source: [Tokio official site](https://tokio.rs/), [Evolution of Async Rust](https://blog.jetbrains.com/rust/2026/02/17/the-evolution-of-async-rust-from-tokio-to-high-level-applications/)

Tokio is production-proven — used by Cloudflare, Discord, AWS Lambda, and many other services. It provides "zero-cost abstractions giving bare-metal performance."

### 3.3 Cross-Compilation & Distribution

Source: [Rust cross-compilation](https://users.rust-lang.org/t/how-does-golangs-cross-compilation-differ-from-rusts/9014), [Rust vs Go deployment](https://users.rust-lang.org/t/rust-vs-golang-deployment-statically-linked-binaries/68760)

| Aspect              | Rust                              | Go                                |
| ------------------- | --------------------------------- | --------------------------------- |
| Cross-compilation   | Via `cross` tool (Docker-based)   | Built-in: `GOOS=linux GOARCH=arm64 go build` |
| Static binary       | Possible with musl, not default   | Default behavior                  |
| Binary size          | ~500KB (optimized) to 2.1MB       | ~2.5MB typical                    |
| Docker images        | Scratch-based possible (bore does this) | Scratch-based (common pattern) |
| Package managers     | Cargo install, Homebrew, AUR      | Go install, Homebrew              |

**Go wins on cross-compilation simplicity.** Rust can achieve smaller binaries but requires more setup for static linking and cross-platform builds.

### 3.4 Code Complexity

Source: [bore GitHub](https://github.com/ekzhang/bore), [Building HTTPS Tunnel in Go](https://nexo.sh/posts/building-your-own-https-tunnel/)

- bore (Rust): ~400 lines for a minimal TCP tunnel.
- Go HTTPS tunnel: ~300 lines for a minimal HTTPS tunnel with yamux multiplexing.
- Both languages can produce concise tunnel implementations. Go is slightly more terse for networking code.

---

## 4. Existing Projects as Evidence

### Rust Tunnel Projects

| Project    | Stars  | Binary Size | Key Differentiator                      |
| ---------- | ------ | ----------- | --------------------------------------- |
| rathole    | 10k+   | ~500KB      | 2x throughput over frp, minimal memory  |
| bore       | 8k+    | Small       | 400 lines, dead simple                  |
| wstunnel   | 7k+    | N/A         | WebSocket focus, protocol bypass        |
| tunnelto   | 900+   | N/A         | HTTP tunnel with subdomains             |
| ExposeME   | New    | N/A         | tokio + hyper 1.x + rustls + Let's Encrypt |

### Go Tunnel Projects

| Project    | Stars  | Key Differentiator                      |
| ---------- | ------ | --------------------------------------- |
| frp        | 100k+  | Most feature-complete, HTTP/TCP/UDP/QUIC/P2P |
| chisel     | 13k+   | SSH-based, auto LetsEncrypt             |
| sish       | 4k+    | SSH-based with WebSocket                |
| piko       | 2k+    | Designed for production traffic         |
| Pangolin   | 5k+    | Full management UI, IAM, SSL            |

---

## 5. Recommendation: Rust

### Why Rust Over Go

**1. Performance ceiling is higher where it matters for tunnels.**

Tunnel software is I/O-bound proxy code — exactly where Rust's zero-cost abstractions and no-GC architecture provide measurable advantages. rathole's benchmarks prove this empirically: same latency at normal load, but Rust pulls ahead under stress (4000+ QPS) where Go starts failing.

**2. Memory efficiency is critical for tunnel servers.**

A tunnel server holds thousands of concurrent long-lived connections. Rust's deterministic memory management (no GC pauses) means predictable tail latency. Go's GC causes periodic CPU spikes especially under heavy load — exactly when a tunnel server needs to be most responsive.

**3. The Rust async networking ecosystem is mature.**

The stack is production-proven:
- **tokio** — async runtime (powers Cloudflare, Discord, AWS services)
- **hyper** — HTTP implementation
- **rustls** — TLS without OpenSSL dependency (simpler cross-compilation)
- **quinn** — QUIC protocol (future-proof transport)
- **tokio-tungstenite** — WebSocket support

**4. Binary size advantage matters for client distribution.**

rathole achieves ~500KB binaries. For a CLI tool that users download and install, smaller is better — especially for IoT/embedded use cases.

**5. Proven by existing projects.**

rathole, bore, and wstunnel demonstrate that Rust tunnel projects can be successful, well-maintained, and performant. rathole's 2x throughput improvement over frp is empirical proof.

### The Trade-Off

Go would let you ship a working prototype **faster** due to:
- Simpler concurrency model (goroutines are more intuitive than async/await + tokio).
- Easier cross-compilation (built-in vs Docker-based `cross` tool).
- Larger ecosystem of tunnel projects to reference.
- Faster compilation times.

**If time-to-market matters more than performance ceiling, Go is the pragmatic choice.** But for a project meant to be a competitive, long-term ngrok alternative, Rust's performance advantages in the exact domain of tunnel software (high-concurrency, I/O-bound, long-lived connections) make it the stronger choice.

---

## 6. Recommended Rust Tech Stack

| Component          | Crate                | Why                                          |
| ------------------ | -------------------- | -------------------------------------------- |
| Async runtime      | `tokio`              | Industry standard, battle-tested             |
| HTTP server        | `axum`               | Built on hyper, ergonomic for APIs           |
| HTTP client/proxy  | `hyper`              | Low-level control for proxying               |
| TLS                | `rustls`             | Pure Rust, no OpenSSL dep, simpler cross-compile |
| WebSocket          | `tokio-tungstenite`  | For tunnel transport through restrictive networks |
| QUIC (future)      | `quinn`              | Head-of-line blocking elimination            |
| Multiplexing       | `yamux` or custom    | Stream multiplexing over single connection   |
| CLI                | `clap`               | Standard Rust CLI framework                  |
| Config             | `toml` + `serde`     | rathole uses TOML, well-proven for this      |
| Certificates       | `rcgen` + ACME client | Auto Let's Encrypt provisioning             |
| Logging            | `tracing`            | Async-aware structured logging (tokio ecosystem) |
| Serialization      | `serde` + `bincode` or `postcard` | Efficient binary protocol messages |
| Noise encryption   | `snow`               | Noise Protocol (rathole uses this as TLS alternative) |
