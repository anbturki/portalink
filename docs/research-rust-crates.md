# Rust Crate Research for Tunneling Tool

**Date:** 2026-03-04
**Purpose:** Evaluate the latest stable versions of Rust crates for building a tunneling/proxy tool. All versions verified against crates.io and docs.rs.

---

## Table of Contents

1. [Rust Edition Recommendation](#rust-edition-recommendation)
2. [Core Crates](#core-crates)
   - [tokio](#tokio---async-runtime)
   - [hyper](#hyper---http-library)
   - [axum](#axum---http-framework)
   - [rustls](#rustls---tls)
   - [quinn](#quinn---quic)
   - [tokio-tungstenite](#tokio-tungstenite---websocket)
   - [yamux](#yamux---stream-multiplexing)
   - [clap](#clap---cli-argument-parsing)
   - [serde](#serde---serialization-framework)
   - [toml](#toml---toml-configuration)
   - [tracing](#tracing---structured-logging)
   - [rcgen](#rcgen---certificate-generation)
   - [snow](#snow---noise-protocol)
   - [postcard](#postcard---binary-serialization)
   - [bincode (status warning)](#bincode---binary-serialization-unmaintained)
3. [ACME / Let's Encrypt Crates](#acme--lets-encrypt-crates)
4. [Compatibility Matrix](#compatibility-matrix)
5. [MSRV Summary](#msrv-summary)
6. [Recommended Cargo.toml](#recommended-cargotoml)
7. [References](#references)

---

## Rust Edition Recommendation

**Recommendation: Use Rust Edition 2024.**

- Rust Edition 2024 was stabilized with **Rust 1.85.0**, released 2025-02-20.
- `cargo new` and `cargo init` now default to `edition = "2024"`.
- The official Rust Book has been rewritten using 2024 edition idioms.
- All major crates in this stack compile cleanly under the 2024 edition.
- Key improvements in the 2024 edition:
  - `unsafe_op_in_unsafe_fn` lint is now on by default (better safety auditing).
  - Improved `impl Trait` lifetime capture rules (RPIT changes).
  - Refined pattern matching rules.
  - Various ergonomic improvements.
- There is **no reason to start a new project on edition 2021** unless you have a hard dependency that has not been updated. None of the crates in this stack require edition 2021.
- Migration from 2021 to 2024 is automated via `cargo fix --edition`.

**Source:** [Announcing Rust 1.85.0 and Rust 2024](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/) | [Rust 2024 Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2024/index.html)

---

## Core Crates

### tokio - Async Runtime

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **1.49.0** |
| **License** | MIT |
| **MSRV** | 1.70 (LTS), rolling for latest |
| **Downloads** | 1.1B+ total |
| **crates.io** | https://crates.io/crates/tokio |
| **Repository** | https://github.com/tokio-rs/tokio |

**Key Features for Tunnel/Proxy Software:**
- Multi-threaded, work-stealing task scheduler.
- Reactor backed by OS event queue (epoll on Linux, kqueue on macOS, IOCP on Windows).
- Asynchronous TCP and UDP sockets.
- Timers, signal handling, filesystem operations.
- `features = ["full"]` enables: `rt-multi-thread`, `net`, `io-util`, `time`, `sync`, `signal`, `fs`, `macros`.

**Notable in v1.49.0:**
- IPv6 TCLASS option support on net sockets.
- Stabilized `runtime::id::Id`.
- `Extend` impl for `JoinSet`.

**LTS Releases:**
- **1.43.x** -- LTS until March 2026, MSRV 1.70.
- **1.47.x** -- LTS until September 2026, MSRV 1.70.

**Known Issues/Limitations:**
- MSRV policy: supports Rust versions at least 6 months old (rolling for non-LTS).
- `features = ["full"]` pulls everything; for minimal binary size, select individual features.

**Compatibility:** Foundation of the entire async stack. hyper, axum, quinn, tokio-tungstenite, and tracing all depend on tokio.

---

### hyper - HTTP Library

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **1.7.0** |
| **License** | MIT |
| **MSRV** | 1.63 |
| **Downloads** | 400M+ total |
| **crates.io** | https://crates.io/crates/hyper |
| **Repository** | https://github.com/hyperium/hyper |

**Companion crate:** `hyper-util` **0.1.20** (higher-level utilities for hyper 1.x).

**Key Features for Tunnel/Proxy Software:**
- Low-level HTTP/1.1 and HTTP/2 implementation.
- Both client and server APIs.
- HTTP CONNECT method support -- essential for HTTP tunneling.
- Streaming request/response bodies via `http-body` trait.
- Full control over connection lifecycle (important for proxy keep-alive).
- Tower-service compatible.

**Known Issues/Limitations:**
- hyper 1.x is deliberately more modular and lower-level than 0.14.x. You typically need `hyper-util` for convenience wrappers.
- No built-in TLS. Must pair with `hyper-rustls` or `hyper-tls` for HTTPS.
- MSRV policy: 6-month rolling window.

**Compatibility:** axum is built on hyper 1.x. Works with tokio runtime. Use `hyper-rustls` (latest: 0.27.x) for TLS integration with rustls.

---

### axum - HTTP Framework

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.8.8** |
| **License** | MIT |
| **MSRV** | ~1.75 (inferred from ecosystem; follows tokio-rs policy) |
| **Downloads** | 75M+ total |
| **crates.io** | https://crates.io/crates/axum |
| **Repository** | https://github.com/tokio-rs/axum |

**Key Features for Tunnel/Proxy Software:**
- Ergonomic routing and Tower middleware system.
- WebSocket upgrade support via `axum::extract::ws`.
- Extractors for headers, query params, JSON, state.
- Tower middleware compatibility (timeouts, rate limiting, compression, CORS).
- Graceful shutdown support.
- State sharing across handlers via `axum::extract::State`.

**Known Issues/Limitations:**
- Still at 0.x versioning; API may change between 0.8 and 0.9 (though 0.8 is stable in practice).
- For pure proxy/tunnel data plane, you may only need hyper directly. axum is best suited for the **control plane / management API**.

**Compatibility:** Built on hyper 1.x, tokio, and tower. Part of the tokio-rs ecosystem. Seamless integration with tracing.

---

### rustls - TLS

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.23.36** (released 2026-01-05) |
| **License** | Apache-2.0 OR MIT OR ISC (triple-licensed) |
| **MSRV** | 1.71 (1.75 with optional zlib-rs) |
| **Downloads** | 300M+ total |
| **crates.io** | https://crates.io/crates/rustls |
| **Repository** | https://github.com/rustls/rustls |

**Companion crate:** `tokio-rustls` **0.26.4** (async TLS streams for tokio).

**Key Features for Tunnel/Proxy Software:**
- Pure Rust TLS 1.2 and TLS 1.3 implementation.
- No dependency on OpenSSL -- dramatically simplifies cross-compilation and reduces supply chain risk.
- Pluggable crypto backends: `ring` (default, smaller) or `aws-lc-rs` (FIPS-capable).
- Certificate verification via `webpki`.
- SNI (Server Name Indication) support -- critical for multi-tenant tunnel servers.
- ALPN negotiation -- needed for HTTP/2 and QUIC.
- Session resumption and 0-RTT.
- Audited by multiple security firms.

**Known Issues/Limitations:**
- Does **not** support TLS 1.0/1.1 by design (these are deprecated protocols).
- No DTLS support (use quinn/QUIC instead).
- `aws-lc-rs` backend requires a C compiler; `ring` includes assembly but is easier to cross-compile.

**Compatibility:** Used by quinn (QUIC/TLS), hyper-rustls (HTTPS), tokio-rustls (async TLS), and rcgen (certificate generation). Central to the TLS stack.

---

### quinn - QUIC

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.11.9** |
| **License** | MIT OR Apache-2.0 |
| **MSRV** | 1.80 |
| **Downloads** | 86M+ total |
| **crates.io** | https://crates.io/crates/quinn |
| **Repository** | https://github.com/quinn-rs/quinn |

**Key Features for Tunnel/Proxy Software:**
- Pure Rust async QUIC implementation (IETF RFC 9000).
- **Built-in stream multiplexing** -- eliminates need for yamux when using QUIC transport.
- 0-RTT connection establishment -- faster reconnection.
- Connection migration -- client can switch networks without dropping tunnel.
- Built on rustls 0.23 for TLS 1.3.
- Supports both `ring` and `aws-lc-rs` crypto providers.
- UDP-based -- better performance on lossy/high-latency networks than TCP.

**Known Issues/Limitations:**
- Still at 0.x; API continues to evolve between minor versions.
- QUIC requires UDP port access; some corporate firewalls and networks block UDP.
- Higher CPU overhead than raw TCP for simple tunneling scenarios.
- MSRV policy: minimum 6 months old Rust version.

**Compatibility:** Uses rustls 0.23.x for TLS. Requires tokio runtime. **quinn 0.11 is confirmed compatible with rustls 0.23**. Recently upgraded to `rustls-platform-verifier` 0.5.x to fix build issues with rustls 0.23.24+.

---

### tokio-tungstenite - WebSocket

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.28.0** |
| **License** | MIT |
| **MSRV** | 1.63 |
| **Downloads** | 80M+ total |
| **crates.io** | https://crates.io/crates/tokio-tungstenite |
| **Repository** | https://github.com/snapview/tokio-tungstenite |

**Key Features for Tunnel/Proxy Software:**
- Async WebSocket client and server on tokio.
- TLS support via feature flags: `native-tls`, `rustls-tls-native-roots`, `rustls-tls-webpki-roots`.
- Streaming message interface (AsyncRead/AsyncWrite).
- WebSocket upgrade from HTTP connections.
- Binary and text frame support.
- Versions > 0.26.2 have significant performance improvements (on par with fastwebsockets).

**Known Issues/Limitations:**
- WebSocket framing adds overhead compared to raw TCP.
- No built-in multiplexing; must layer yamux on top for multi-stream support.

**Compatibility:** Built on tokio. TLS via rustls feature flags. Note: axum has its own WebSocket support via `axum::extract::ws` (based on tungstenite internally). Choose one approach -- either axum's built-in WS or tokio-tungstenite directly.

---

### yamux - Stream Multiplexing

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.13.8** |
| **License** | Apache-2.0 OR MIT |
| **MSRV** | ~1.65 (not explicitly documented) |
| **Downloads** | 20M+ total |
| **crates.io** | https://crates.io/crates/yamux |
| **Repository** | https://github.com/libp2p/rust-yamux |

**Key Features for Tunnel/Proxy Software:**
- Yamux specification implementation for multiplexing independent I/O streams over a single connection.
- Works over any reliable, ordered transport (TCP, TLS, WebSocket).
- Flow control with backpressure.
- `Connection` type wraps underlying I/O; provides `open_stream()` and `accept()`.
- `Stream` implements `futures::io::AsyncRead` and `futures::io::AsyncWrite`.
- Lightweight frame headers (12 bytes per frame).

**Known Issues/Limitations:**
- Maintained by Parity Technologies (libp2p ecosystem); **not part of tokio-rs**.
- Uses `futures::io` traits, **not** `tokio::io` traits. Requires `tokio-util::compat` for interop with tokio I/O.
- If using QUIC (quinn), QUIC's built-in stream multiplexing eliminates the need for yamux.

**Alternatives:**
- `tokio-yamux` (latest 0.3.16) -- tokio-native yamux wrapper, avoids the compat layer.
- QUIC's built-in multiplexing via quinn -- preferred when using QUIC transport.

**Compatibility:** Independent of the tokio/hyper/axum stack. Uses `futures` crate traits. Interop with tokio requires `tokio-util` compatibility layer or use `tokio-yamux` instead.

---

### clap - CLI Argument Parsing

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **4.5.59** |
| **License** | MIT OR Apache-2.0 |
| **MSRV** | 1.74 |
| **Downloads** | 400M+ total |
| **crates.io** | https://crates.io/crates/clap |
| **Repository** | https://github.com/clap-rs/clap |

**Key Features for Tunnel/Proxy Software:**
- Derive macro for declarative CLI definition: `#[derive(Parser)]`.
- Subcommands (e.g., `tunnel server`, `tunnel client`, `tunnel config`).
- Environment variable fallback (`#[arg(env = "TUNNEL_PORT")]`).
- Config file integration patterns.
- Shell completion generation via `clap_complete`.
- Colored help output.

**Known Issues/Limitations:**
- MSRV follows "N-2" policy (last two stable Rust releases); bumps relatively often.
- Binary size impact: `clap` with derive adds ~200-400KB. Use builder API for smaller binaries.

**Compatibility:** Standalone; no async runtime dependency. Works with serde for config file loading.

---

### serde - Serialization Framework

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **1.0.228** |
| **License** | MIT OR Apache-2.0 |
| **MSRV** | ~1.31 (historically very conservative) |
| **Downloads** | 699M+ total |
| **crates.io** | https://crates.io/crates/serde |
| **Repository** | https://github.com/serde-rs/serde |

**Key Features for Tunnel/Proxy Software:**
- Universal serialization/deserialization framework for Rust.
- Derive macros: `#[derive(Serialize, Deserialize)]`.
- Used by virtually every config and data format crate (toml, json, postcard).
- `no_std` support available.
- Zero-copy deserialization for borrowed data.
- The most battle-tested serialization library in the Rust ecosystem.

**Known Issues/Limitations:**
- The `derive` feature must be explicitly enabled: `serde = { version = "1", features = ["derive"] }`.
- Stable at 1.x; no breaking changes expected.

**Compatibility:** Required by toml, postcard, clap (optionally), and many other crates. Universal dependency across this entire stack.

---

### toml - TOML Configuration

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.9.8** |
| **License** | MIT OR Apache-2.0 |
| **MSRV** | Not explicitly documented for 0.9 |
| **Downloads** | 416M+ total |
| **crates.io** | https://crates.io/crates/toml |
| **Repository** | https://github.com/toml-rs/toml |

**Key Features for Tunnel/Proxy Software:**
- Native Rust TOML encoder and decoder.
- Full serde integration (Serialize/Deserialize traits).
- Feature flags: `parse`, `display`, `default`, `debug`, `fast_hash`.
- Built on `toml_edit` internally for robust parsing.

**Known Issues/Limitations:**
- Still at 0.x versioning.
- For preserving comments and formatting when editing config files programmatically, consider using `toml_edit` directly.

**Compatibility:** Depends on serde. Works naturally with clap for config-file-plus-CLI-args patterns.

---

### tracing - Structured Logging

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.1.41** |
| **License** | MIT |
| **MSRV** | 1.63 |
| **Downloads** | 387M+ total |
| **crates.io** | https://crates.io/crates/tracing |
| **Repository** | https://github.com/tokio-rs/tracing |

**Companion crate:** `tracing-subscriber` **0.3.20**.

**Key Features for Tunnel/Proxy Software:**
- Structured, async-aware logging and diagnostics.
- Span-based context propagation -- track requests through the entire proxy pipeline.
- Multiple subscriber backends: console (`tracing-subscriber`), file (`tracing-appender`), JSON.
- Compatible with `log` crate via `tracing-log` bridge.
- Performance-oriented: spans can be compiled out in release builds.
- `#[instrument]` attribute macro for automatic span creation on functions.

**Known Issues/Limitations:**
- Still at 0.1.x despite being extremely mature and widely used.
- `tracing-subscriber` is a separate crate; you need both.

**Compatibility:** Part of the tokio-rs ecosystem. First-class integration with tokio, hyper, and axum. axum has built-in tracing middleware.

---

### rcgen - Certificate Generation

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.14.6** (per docs.rs latest) |
| **License** | MIT OR Apache-2.0 |
| **MSRV** | 1.88 (workspace Cargo.toml; policy: support ~7-month-old Rust) |
| **Downloads** | 85M+ total |
| **crates.io** | https://crates.io/crates/rcgen |
| **Repository** | https://github.com/rustls/rcgen |

**Key Features for Tunnel/Proxy Software:**
- Generate X.509 certificates and CSRs in pure Rust.
- Self-signed certificate generation (development / internal tunnel CA).
- CA certificate creation for signing client/server certificates.
- Pluggable crypto backends: `ring` or `aws-lc-rs`.
- FIPS mode via `aws-lc-rs` + `fips` feature.
- Can build without any cryptography provider (`crypto` feature is optional).
- `generate_simple_self_signed()` for quick setup; `CertificateParams` for full control.

**Known Issues/Limitations:**
- **MSRV of 1.88 is the highest in this stack** -- sets the floor for the project's Rust toolchain.
- Part of the rustls project; API changes track rustls release cycles.
- 0.14 had significant API changes: `signed_by()` methods now take `&Issuer` reference.

**Compatibility:** Part of the rustls ecosystem. Generated certificates are directly usable with rustls `ServerConfig` / `ClientConfig`. Shares crypto backend choices with rustls and quinn.

---

### snow - Noise Protocol

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.10.0** |
| **License** | Apache-2.0 OR MIT |
| **MSRV** | Not explicitly documented |
| **Downloads** | 17.5M+ total |
| **crates.io** | https://crates.io/crates/snow |
| **Repository** | https://github.com/mcginty/snow |

**Key Features for Tunnel/Proxy Software:**
- Pure Rust Noise Protocol Framework implementation.
- Tracks Noise spec revision 34 (latest).
- Supported DHs: Curve25519, P-256 (non-standard extension).
- Supported Ciphers: AES-GCM, ChaChaPoly, XChaChaPoly (non-standard).
- Supported Hashes: SHA-256, SHA-512, BLAKE2s, BLAKE2b.
- Pluggable crypto resolvers via `Builder::with_resolver()`.
- `no_std` support (with `alloc`, default resolver only).
- Useful for custom tunnel authentication and key exchange without TLS overhead.

**Known Issues/Limitations:**
- Fallback modifier is **not implemented**.
- XChaChaPoly and P-256 are non-standard Noise extensions; interop with other implementations requires care.
- Less commonly used than TLS; requires both sides to implement Noise.

**Compatibility:** Independent of the tokio/rustls stack. Can be layered on any transport (TCP, UDP, WebSocket). Provides its own key exchange mechanism; use **instead of** TLS for custom tunnel protocols, or as an additional layer.

---

### postcard - Binary Serialization

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **1.1.3** |
| **License** | MIT OR Apache-2.0 |
| **MSRV** | Not explicitly documented |
| **Downloads** | 22M+ total |
| **crates.io** | https://crates.io/crates/postcard |
| **Repository** | https://github.com/jamesmunns/postcard |

**Key Features for Tunnel/Proxy Software:**
- `no_std` focused serde-compatible serializer/deserializer.
- Compact wire format using variable-length integer encoding (varint).
- **Documented and stable wire format since v1.0.0** -- critical for protocol stability.
- Very fast serialization/deserialization.
- Feature flags: `alloc`, `use-std`, `heapless-cas` (default), `embedded-io`, `crc`.
- Ideal for control plane messages between tunnel client and server.
- Small binary size footprint.

**Known Issues/Limitations:**
- No schema evolution support (no field names in wire format). Both sides must agree on schema.
- Not self-describing; unsuitable for human-readable debugging without tooling.
- `postcard2` exists as a separate future major version.

**Compatibility:** Depends on serde. Transport-agnostic; works with any I/O layer. Recommended replacement for bincode.

---

### bincode - Binary Serialization (UNMAINTAINED)

| Field | Value |
|-------|-------|
| **Last Maintained Version** | 2.0.1 |
| **License** | MIT |
| **Status** | **UNMAINTAINED -- RUSTSEC-2025-0141** |
| **crates.io** | https://crates.io/crates/bincode |

**DO NOT USE FOR NEW PROJECTS.** Bincode development has permanently ceased due to a doxxing and harassment incident against the maintainer. No further releases will be published on crates.io. A RustSec advisory ([RUSTSEC-2025-0141](https://rustsec.org/advisories/RUSTSEC-2025-0141)) was issued in January 2026.

**Recommended alternatives (from the bincode team):**
- **postcard** -- recommended for this project (see above).
- **bitcode** -- compact binary encoding.
- **rkyv** -- zero-copy deserialization.
- **bincode-next** (v2.0.4) -- community fork, if bincode wire format compatibility is needed.

---

## ACME / Let's Encrypt Crates

### instant-acme (Recommended for low-level ACME control)

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.8.4** |
| **License** | Apache-2.0 |
| **MSRV** | 1.70 |
| **crates.io** | https://crates.io/crates/instant-acme |
| **Repository** | https://github.com/instant-labs/instant-acme |

- Async pure-Rust ACME client implementing RFC 8555.
- Built on tokio and rustls.
- Used in production at Instant Domain Search.
- Feature flags: `hyper-rustls` (default), `aws-lc-rs` (default), `ring`, `fips`, `x509-parser`, `time`.
- Full support for ACME Renewal Information (ARI, RFC 9773) added in 0.8.
- Supports all ACME challenge types (HTTP-01, DNS-01, TLS-ALPN-01).

### rustls-acme (Recommended for automatic certificate serving)

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.14.1** |
| **License** | Apache-2.0 OR MIT |
| **crates.io** | https://crates.io/crates/rustls-acme |

- Higher-level than instant-acme: handles certificate provisioning and serving automatically.
- Supports TLS-ALPN-01 and HTTP-01 challenges.
- Can serve ACME challenges and regular TLS traffic on the same port.
- Good for automatic certificate management in a tunnel server.

### tokio-rustls-acme

| Field | Value |
|-------|-------|
| **Latest Stable Version** | **0.9.0** |
| **License** | Apache-2.0 OR MIT |
| **crates.io** | https://crates.io/crates/tokio-rustls-acme |
| **Repository** | https://github.com/n0-computer/tokio-rustls-acme |

- Fork of rustls-acme optimized for the tokio runtime.
- Maintained by n0-computer (Iroh/IPFS ecosystem).
- Good option for tight tokio integration.

**Recommendation:** Use **instant-acme** for full control over the ACME flow, or **rustls-acme** / **tokio-rustls-acme** for automatic certificate management integrated directly into the TLS listener.

---

## Compatibility Matrix

| Crate | Depends On | Compatible With |
|-------|-----------|-----------------|
| **tokio** 1.49 | (foundation) | All crates in this stack |
| **hyper** 1.7 | tokio | axum, rustls (via hyper-rustls) |
| **hyper-util** 0.1.20 | hyper 1.x, tokio | axum, hyper |
| **axum** 0.8 | hyper 1.x, tokio, tower | tokio-tungstenite (WS), tracing |
| **rustls** 0.23 | ring or aws-lc-rs | quinn 0.11, hyper-rustls, tokio-rustls, rcgen |
| **tokio-rustls** 0.26 | rustls 0.23, tokio | hyper, axum |
| **quinn** 0.11 | rustls 0.23, tokio | rustls 0.23 (confirmed compatible) |
| **tokio-tungstenite** 0.28 | tokio, tungstenite | rustls (via feature flag) |
| **yamux** 0.13 | futures | tokio (via tokio-util::compat) |
| **clap** 4.5 | (standalone) | serde (for config derive) |
| **serde** 1.0 | (standalone) | toml, postcard, clap |
| **toml** 0.9 | serde | serde, clap |
| **tracing** 0.1 | tracing-core | tokio, hyper, axum (native) |
| **rcgen** 0.14 | ring or aws-lc-rs | rustls (same ecosystem) |
| **snow** 0.10 | (standalone crypto) | Any transport |
| **postcard** 1.1 | serde | serde |
| **instant-acme** 0.8 | tokio, rustls, hyper | rustls 0.23, tokio |

**Critical compatibility note:** quinn 0.11 and rustls 0.23 are confirmed compatible. When selecting crypto backends, maintain consistency: if you choose `aws-lc-rs` for rustls, use it for quinn and rcgen too (or vice versa with `ring`). Mixing backends will compile but increases binary size and attack surface.

---

## MSRV Summary

The project's minimum Rust version is determined by the crate with the highest MSRV.

| Crate | MSRV | Notes |
|-------|------|-------|
| **rcgen** 0.14 | **1.88** | **Highest -- sets the project floor** |
| **quinn** 0.11 | 1.80 | |
| **axum** 0.8 | ~1.75 | Not explicitly documented; inferred |
| **clap** 4.5 | 1.74 | N-2 Rust release policy |
| **tokio** 1.49 | ~1.71 | 6-month rolling; LTS at 1.70 |
| **rustls** 0.23 | 1.71 | 1.75 with optional zlib-rs |
| **instant-acme** 0.8 | 1.70 | |
| **yamux** 0.13 | ~1.65 | Not explicitly documented |
| **tracing** 0.1 | 1.63 | |
| **hyper** 1.7 | 1.63 | 6-month rolling policy |
| **tokio-tungstenite** 0.28 | 1.63 | |
| **serde** 1.0 | ~1.31 | Very conservative |
| **snow** 0.10 | Not documented | |
| **postcard** 1.1 | Not documented | |
| **toml** 0.9 | Not documented | |

**Project MSRV: Rust 1.88** (due to rcgen 0.14). Use the **latest stable Rust toolchain** to avoid issues.

---

## Recommended Cargo.toml

```toml
[package]
name = "tunnling"
version = "0.1.0"
edition = "2024"
rust-version = "1.88"

[dependencies]
# Async runtime
tokio = { version = "1.49", features = ["full"] }

# HTTP stack
hyper = { version = "1.7", features = ["http1", "http2", "server", "client"] }
hyper-util = { version = "0.1", features = ["full"] }
axum = { version = "0.8", features = ["ws"] }

# TLS
rustls = { version = "0.23", default-features = false, features = ["ring", "logging", "std", "tls12"] }
tokio-rustls = "0.26"

# QUIC (optional transport)
quinn = "0.11"

# WebSocket
tokio-tungstenite = { version = "0.28", features = ["rustls-tls-webpki-roots"] }

# Stream multiplexing (for TCP/TLS/WS transport; not needed over QUIC)
yamux = "0.13"
tokio-util = { version = "0.7", features = ["compat"] }

# CLI
clap = { version = "4.5", features = ["derive", "env"] }

# Configuration
serde = { version = "1", features = ["derive"] }
toml = "0.9"

# Logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Certificate generation
rcgen = "0.14"

# Noise Protocol (for custom authentication)
snow = "0.10"

# Binary serialization (for tunnel control protocol)
postcard = { version = "1.1", features = ["alloc"] }

# ACME / Let's Encrypt
instant-acme = "0.8"
```

### Architecture Notes

1. **Transport Layer:** Use QUIC (quinn) as the primary transport between tunnel server and client. QUIC provides built-in multiplexing, making yamux unnecessary for that path. Keep TCP+TLS+yamux as a fallback for networks that block UDP.

2. **WebSocket Transport:** Use tokio-tungstenite for a WebSocket transport option that traverses HTTP proxies and restrictive firewalls. Layer yamux on top for stream multiplexing.

3. **TLS Strategy:** Use rustls throughout. Choose **one** crypto backend (`ring` or `aws-lc-rs`) and use it consistently across rustls, quinn, and rcgen to avoid duplicated crypto code in the binary.

4. **Serialization:** Use postcard for internal tunnel control messages (compact, fast, serde-compatible, stable wire format). Do **not** use bincode (unmaintained, RUSTSEC advisory).

5. **Certificate Management:** Use rcgen for generating self-signed CA and client/server certificates. Use instant-acme or rustls-acme for automated Let's Encrypt certificate provisioning on the public-facing tunnel server.

6. **Control Plane vs Data Plane:** Use axum for the management/control API (status, metrics, config endpoints). Use hyper directly for the tunnel data plane where low-level HTTP CONNECT handling and raw byte forwarding are needed.

---

## References

### crates.io Pages (version source of truth)
- [tokio](https://crates.io/crates/tokio) -- verified 1.49.0
- [hyper](https://crates.io/crates/hyper) -- verified 1.7.0
- [hyper-util](https://crates.io/crates/hyper-util) -- verified 0.1.20
- [axum](https://crates.io/crates/axum) -- verified 0.8.8
- [rustls](https://crates.io/crates/rustls) -- verified 0.23.36
- [tokio-rustls](https://crates.io/crates/tokio-rustls) -- verified 0.26.4
- [quinn](https://crates.io/crates/quinn) -- verified 0.11.9
- [tokio-tungstenite](https://crates.io/crates/tokio-tungstenite) -- verified 0.28.0
- [yamux](https://crates.io/crates/yamux) -- verified 0.13.8
- [clap](https://crates.io/crates/clap) -- verified 4.5.59
- [serde](https://crates.io/crates/serde) -- verified 1.0.228
- [toml](https://crates.io/crates/toml) -- verified 0.9.8
- [tracing](https://crates.io/crates/tracing) -- verified 0.1.41
- [tracing-subscriber](https://crates.io/crates/tracing-subscriber) -- verified 0.3.20
- [rcgen](https://crates.io/crates/rcgen) -- verified 0.14.6
- [snow](https://crates.io/crates/snow) -- verified 0.10.0
- [postcard](https://crates.io/crates/postcard) -- verified 1.1.3
- [bincode](https://crates.io/crates/bincode) -- 2.0.1 (UNMAINTAINED)
- [instant-acme](https://crates.io/crates/instant-acme) -- verified 0.8.4
- [rustls-acme](https://crates.io/crates/rustls-acme) -- verified 0.14.1
- [tokio-rustls-acme](https://crates.io/crates/tokio-rustls-acme) -- verified 0.9.0

### docs.rs Documentation
- [tokio latest](https://docs.rs/crate/tokio/latest)
- [hyper latest](https://docs.rs/crate/hyper/latest)
- [axum latest](https://docs.rs/crate/axum/latest)
- [rustls latest](https://docs.rs/crate/rustls/latest)
- [quinn latest](https://docs.rs/crate/quinn/latest)
- [tokio-tungstenite latest](https://docs.rs/crate/tokio-tungstenite/latest)
- [yamux latest](https://docs.rs/crate/yamux/latest)
- [clap latest](https://docs.rs/crate/clap/latest)
- [rcgen latest](https://docs.rs/crate/rcgen/latest)
- [snow latest](https://docs.rs/crate/snow/latest)
- [postcard latest](https://docs.rs/crate/postcard/latest)
- [instant-acme latest](https://docs.rs/instant-acme/latest/instant_acme/)

### GitHub Repositories
- [tokio-rs/tokio](https://github.com/tokio-rs/tokio)
- [hyperium/hyper](https://github.com/hyperium/hyper)
- [tokio-rs/axum](https://github.com/tokio-rs/axum)
- [rustls/rustls](https://github.com/rustls/rustls)
- [quinn-rs/quinn](https://github.com/quinn-rs/quinn)
- [snapview/tokio-tungstenite](https://github.com/snapview/tokio-tungstenite)
- [libp2p/rust-yamux](https://github.com/libp2p/rust-yamux)
- [clap-rs/clap](https://github.com/clap-rs/clap)
- [serde-rs/serde](https://github.com/serde-rs/serde)
- [toml-rs/toml](https://github.com/toml-rs/toml)
- [tokio-rs/tracing](https://github.com/tokio-rs/tracing)
- [rustls/rcgen](https://github.com/rustls/rcgen)
- [mcginty/snow](https://github.com/mcginty/snow)
- [jamesmunns/postcard](https://github.com/jamesmunns/postcard)
- [instant-labs/instant-acme](https://github.com/instant-labs/instant-acme)
- [n0-computer/tokio-rustls-acme](https://github.com/n0-computer/tokio-rustls-acme)

### Other Sources
- [Announcing Rust 1.85.0 and Rust 2024 (Rust Blog)](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/)
- [Rust 2024 Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2024/index.html)
- [RUSTSEC-2025-0141: Bincode is unmaintained](https://rustsec.org/advisories/RUSTSEC-2025-0141)
- [hyper MSRV policy](https://hyper.rs/contrib/msrv/)
- [rcgen Cargo.toml (MSRV source)](https://github.com/rustls/rcgen/blob/main/Cargo.toml)
- [quinn Cargo.toml (MSRV source)](https://github.com/quinn-rs/quinn/blob/main/Cargo.toml)
- [lib.rs/crates/instant-acme (features)](https://lib.rs/crates/instant-acme/features)
- [lib.rs/crates/postcard (features)](https://lib.rs/crates/postcard/features)
