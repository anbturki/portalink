# Research: QUIC as a Transport Layer for Tunneling Software in Rust

Date: 2026-03-04

---

## Table of Contents

1. [Quinn Library](#1-quinn-library)
2. [QUIC vs TCP+TLS for Tunnels](#2-quic-vs-tcptls-for-tunnels)
3. [Real QUIC Tunnel Implementations](#3-real-quic-tunnel-implementations)
4. [QUIC Multiplexing vs Yamux](#4-quic-multiplexing-vs-yamux)
5. [Practical Concerns](#5-practical-concerns)
6. [Recommendations](#6-recommendations)
7. [References](#7-references)

---

## 1. Quinn Library

### Overview

Quinn is a pure-Rust, async-compatible implementation of the IETF QUIC transport protocol. It is the most widely used QUIC library in the Rust ecosystem, with 86.1M total downloads on crates.io. The latest stable version is **0.11.9** (released 2025-08-27). The project was founded by Dirkjan Ochtman and Benjamin Saunders in 2018 and has seen more than 30 releases.

Quinn provides reliable, multiplexed streams over UDP with built-in TLS 1.3 encryption, fast connection establishment (0-RTT and 1-RTT), improved congestion control, and connection migration support.

### Crate Architecture

Quinn is split into several crates:

| Crate | Purpose |
|-------|---------|
| `quinn` | User-facing async API (the main entry point) |
| `quinn-proto` | Core protocol state machine (no I/O, no async runtime dependency) |
| `quinn-udp` | Platform-specific UDP I/O operations |

This separation allows `quinn-proto` to be used in `no_std` or custom async runtime environments, while `quinn` provides the ergonomic Tokio-based API most applications need.

### Key API Types and Usage

**Endpoint** -- The entry point. An `Endpoint` binds a UDP socket and manages all connections through it. A single endpoint can function as both client and server simultaneously (relevant for peer-to-peer tunneling).

**Connection** -- Represents an established QUIC connection. Key methods:

- `open_bi()` -- Opens a new bidirectional stream, returns `(SendStream, RecvStream)`
- `open_uni()` -- Opens a new unidirectional stream, returns `SendStream`
- `accept_bi()` -- Accepts an incoming bidirectional stream from the peer
- `accept_uni()` -- Accepts an incoming unidirectional stream
- `remote_address()` -- Returns the peer's current UDP address
- `close()` -- Closes the connection with an error code and reason

**SendStream / RecvStream** -- Lightweight stream handles for sending and receiving data. Streams are cheap to create and disposable.

**Important API behavior**: The side that calls `open_bi()` must write to its `SendStream` before the other side can `accept_bi()`. Calling `open_bi()` and waiting on the `RecvStream` without writing first will deadlock.

### Connection Establishment Example Pattern

```rust
// Server side
let endpoint = Endpoint::server(server_config, "0.0.0.0:4433".parse()?)?;
while let Some(connecting) = endpoint.accept().await {
    let connection = connecting.await?;
    // Each new QUIC connection can spawn multiple streams
    tokio::spawn(async move {
        while let Ok((send, recv)) = connection.accept_bi().await {
            // Handle each stream independently
            tokio::spawn(handle_stream(send, recv));
        }
    });
}

// Client side
let endpoint = Endpoint::client("0.0.0.0:0".parse()?)?;
let connection = endpoint.connect(server_addr, "server-hostname")?.await?;
let (send, recv) = connection.open_bi().await?;
// Use send/recv for data transfer
```

### Multiplexing

QUIC natively supports multiple independent streams within a single connection. Each stream is managed separately -- if a packet is lost, it only affects the specific stream carrying that data, not other streams on the same connection. Streams can be opened up to a limit dictated by the peer, and that limit is typically increased as old streams are completed.

For tunneling, this means each proxied connection (e.g., an HTTP request from a user) can be mapped to its own QUIC stream, achieving true multiplexing without additional protocol layers.

### TLS Integration

Quinn **requires** TLS 1.3. This is mandated by the QUIC specification itself (RFC 9001), not just a Quinn design choice. The primary TLS implementation used is **rustls**, with support for two crypto providers:

- **ring** (default) -- Pure Rust, well-audited
- **aws-lc-rs** -- AWS's crypto library, potentially faster for some operations

There is no option to disable TLS in QUIC. This is a security feature of the protocol -- all QUIC connections are encrypted by default. For tunneling software, this eliminates the need for a separate TLS layer (unlike TCP tunnels that must add TLS on top).

Configurable cipher suites in the ecosystem include:
- `chacha20-poly1305` (good for platforms without AES-NI hardware acceleration)
- `aes-256-gcm`
- `aes-128-gcm`

### Connection Migration

Quinn supports QUIC connection migration, which allows a connection to survive network changes (e.g., switching from Wi-Fi to cellular). Key API points:

- **Server-side**: Set `ServerConfig::migration(true)` to allow clients to change addresses freely
- **Client-side**: `Endpoint::rebind()` updates the endpoint's bound address live, affecting all active connections. Incoming connections and connections to servers unreachable from the new address will be lost.
- QUIC uses a Connection ID (not the IP/port tuple) to identify connections, so address changes are transparent to the application layer

For tunneling, connection migration means the tunnel agent on a laptop can switch networks without dropping the tunnel or requiring reconnection -- a significant advantage over TCP-based tunnels.

### 0-RTT Connection Establishment

Quinn supports 0-RTT (zero round-trip time) connection resumption for reconnecting to previously contacted servers. When a client has cached TLS session information from a prior connection, it can send application data in the very first packet to the server, eliminating the full handshake latency.

**API**: The `Connecting` type provides a method that returns `Ok` immediately if 0-RTT data can be sent. The returned `Connection` can be used to send data before the handshake completes.

**Security considerations for 0-RTT**:

- **Replay attacks**: 0-RTT data is vulnerable to replay attacks. An attacker can capture and replay the initial packet, causing the server to process the same data multiple times. Quinn's documentation explicitly warns: "0-RTT data should therefore never invoke non-idempotent operations."
- **Forward secrecy**: 0-RTT data does not have full forward secrecy guarantees -- it uses keys derived from the previous session's state.
- **Mitigation**: Applications must ensure 0-RTT data is idempotent. For tunneling, this means the initial handshake/authentication exchange should not be sent via 0-RTT, but subsequent stream opens on an already-authenticated tunnel could potentially benefit.
- RFC 9001 states: "Endpoints must implement and use the replay protections described in TLS 1.3, however it is recognized that these protections are imperfect."

**Practical impact for tunneling**: The tunnel agent reconnecting to the relay server after a brief disconnection can resume with 0-RTT, sending its first tunneled packet immediately rather than waiting for a full handshake. This reduces reconnection latency from ~2-3 RTTs (TCP+TLS) to 0 RTTs.

### Performance Characteristics

Based on benchmark studies (ETH Zurich semester thesis, 2024; various QUIC benchmark papers):

- **Throughput**: Quinn achieves up to **8.22 Gbit/s** average throughput in ideal conditions, the fastest among tested QUIC implementations. MsQuic (Microsoft's C implementation) follows at 5.81 Gbit/s.
- **Segment size tuning**: Increasing segment sizes for Quinn nearly doubled throughput and reduced system calls in benchmark tests.
- **Loss conditions**: Under packet loss or reordering, MsQuic outperforms Quinn by at least 50%. Quinn's loss recovery is functional but less optimized than MsQuic's for challenging network conditions.
- **CPU overhead**: QUIC running in userspace introduces higher CPU usage than kernel-space TCP. Data copying overhead between kernel and userspace increases by 26% as packet loss rises from 0% to 2.5%, whereas TCP experiences no additional overhead in this scenario.
- **Kernel bypass potential**: AF_XDP integration has shown 5-40% improvement in requests per second for QUIC implementations, with 5-50% reduction in CPU usage.

---

## 2. QUIC vs TCP+TLS for Tunnels

### Head-of-Line (HOL) Blocking Elimination

**The TCP problem**: When tunneling multiple logical connections over a single TCP connection (as most tunnel implementations do), a single lost packet blocks *all* streams until that packet is retransmitted. This is because TCP provides a single, ordered byte stream -- even if the lost packet belongs to stream A, streams B, C, and D are all stalled.

**How QUIC solves it**: QUIC natively supports multiple independent streams within a single connection. Each stream has its own ordering and flow control. If a packet carrying data for stream A is lost, only stream A is blocked. Streams B, C, and D continue to make progress.

**Impact on tunneling specifically**: Consider a tunnel carrying 50 concurrent HTTP connections. With TCP+yamux, a single lost packet can stall all 50 connections. With QUIC, only the one affected connection stalls. Under even modest packet loss (1-2%), this difference is dramatic for user-perceived latency and responsiveness.

### Connection Establishment Latency

| Protocol | Round Trips | Time (100ms RTT) |
|----------|-------------|-------------------|
| TCP 3-way handshake | 1 RTT | 100ms |
| TCP + TLS 1.3 | 2 RTT | 200ms |
| TCP + TLS 1.2 | 3 RTT | 300ms |
| QUIC (1-RTT, first connection) | 1 RTT | 100ms |
| QUIC (0-RTT, reconnection) | 0 RTT | ~0ms |

QUIC merges the transport handshake and the cryptographic handshake into a single exchange. For a first-time connection, QUIC completes in 1 RTT versus TCP+TLS 1.3's 2 RTTs. For reconnections with cached session state, QUIC achieves 0-RTT -- the client can send data in the very first packet.

For tunneling, this means:
- **Initial tunnel setup** saves 1 RTT compared to TCP+TLS
- **Tunnel reconnection** (after network change, brief disconnection) is nearly instantaneous with 0-RTT

### NAT Traversal Behavior

**UDP vs TCP for NAT traversal**:
- UDP NAT bindings typically expire faster than TCP (30-120 seconds for UDP vs minutes-hours for TCP), requiring keepalive packets
- However, QUIC's connection ID mechanism means the connection survives NAT rebinding -- if the NAT assigns a new port, the server can still identify the connection by its Connection ID rather than the source IP:port tuple
- QUIC's connection migration feature explicitly handles address changes, including those caused by NAT rebinding
- For hole-punching in peer-to-peer scenarios, UDP-based QUIC has better success rates than TCP, as noted by libp2p documentation: "QUIC should be preferred over TCP whenever possible, as it increases the chances of a successful holepunch"

### Firewall and Corporate Proxy Compatibility

**UDP blocking in practice**:
- Google's measurements found that "only a small number of connections" are blocked when using QUIC over UDP on the general internet
- Chrome uses QUIC for "more than half of all connections to Google's servers," indicating broad UDP reachability
- However, many enterprise firewalls and security appliances explicitly block or rate-limit UDP traffic. Products like Palo Alto Networks, Check Point, and Zscaler provide documented procedures for blocking QUIC specifically
- Enterprise security teams block QUIC because it prevents SSL/TLS inspection -- with QUIC, traffic cannot be decrypted by forward proxy appliances in the same way as TCP+TLS

**Key risk for a tunneling product**: Enterprise users behind corporate firewalls are a significant market segment for tunneling software. If the tunnel only supports QUIC, it will fail silently in these environments. A TCP/WebSocket fallback is essential (see Section 5).

### TCP over QUIC Tunnel Performance (Research Paper)

The paper "Implementation and Performance Evaluation of TCP over QUIC Tunnels" (arXiv:2504.10054) provides direct experimental evidence relevant to building a QUIC-based tunnel:

**Key findings**:

1. **Lossy networks**: TCP over QUIC maintains significantly higher throughput than native TCP under packet loss, with major improvement under 20% loss. QUIC's loss recovery mechanisms outperform TCP's in challenging conditions.

2. **Ideal conditions**: In high-quality network environments, the dual-protocol stack introduces additional computational overhead and latency due to userspace processing, encryption, and protocol encapsulation. This leads to lower throughput and higher CPU usage compared to direct TCP.

3. **The trade-off**: The paper establishes a clear crossover point -- QUIC tunneling excels when network conditions are poor (packet loss >1%, high latency, reordering), but incurs overhead penalties in stable, high-speed LAN-like environments.

4. **Implementation**: The paper used Quinn for its tunnel implementation, validating Quinn as a suitable choice for production QUIC tunneling.

**Implications for our tunnel design**: The relay server (running in a data center with stable, low-loss connectivity) will see a small throughput overhead from QUIC. However, the tunnel agent (running on user machines with potentially poor Wi-Fi, mobile, or congested networks) will benefit significantly from QUIC's resilience.

---

## 3. Real QUIC Tunnel Implementations

### rstun (Rust, Quinn-based)

**Repository**: github.com/neevek/rstun

**Architecture**:
- Client binary (`rstunc`) and server binary (`rstund`)
- Built on Quinn for all QUIC transport
- Supports both TCP and UDP traffic tunneling over QUIC
- Multiple tunnels can run simultaneously in a single client/server instance via `--tcp-mappings` and `--udp-mappings` options

**How it uses Quinn**:
- Each tunneled connection maps to a QUIC stream
- Uses Quinn's bidirectional streams for TCP tunnel traffic
- TLS 1.3 with configurable cipher suites: `chacha20-poly1305` (default), `aes-256-gcm`, `aes-128-gcm`

**Connection migration feature**: rstun implements an innovative anti-throttling mechanism using QUIC connection migration. The `--hop-interval-ms` parameter enables periodic migration to a new random local UDP port at configurable intervals. This avoids UDP throttling during long data transfers while maintaining the upper-layer QUIC connection without interruption. This is a practical, production-validated use of Quinn's connection migration API.

**Modes**:
- **OUT mode**: Outbound tunneling (client initiates, traffic exits at server)
- **IN mode**: Inbound tunneling (reverse tunnel, server pushes traffic to client)

**Relevance**: rstun demonstrates that Quinn is suitable for building a production tunnel. Its connection migration for anti-throttling is a technique worth adopting.

### frp QUIC Mode (Go, quic-go)

**Repository**: github.com/fatedier/frp

frp is the most popular open-source reverse proxy/tunnel, with 90k+ GitHub stars.

**QUIC implementation**:
- QUIC added alongside existing TCP transport, not as a replacement
- Uses the `quic-go` library (Go's equivalent of Quinn)
- Configuration is a simple protocol switch in the client config:
  ```toml
  serverAddr = "x.x.x.x"
  serverPort = 7000
  transport.protocol = "quic"
  ```
- The `quicBindPort` on the server can be the same as `bindPort` since they use different L4 protocols (UDP vs TCP)

**Multiplexing approach**: frp already had its own TCP stream multiplexing (since v0.10.0, HTTP/2-style). When using QUIC mode, frp leverages QUIC's native stream multiplexing instead. The connection between client and server supports multiplexing without requiring a new connection per user request, reducing establishment latency and file descriptor usage.

**Relevance**: frp validates the pattern of offering QUIC as a transport option alongside TCP, with the same tunnel protocol running over either. This dual-transport strategy is the recommended approach.

### Reverst (Go, QUIC + HTTP/3)

**Repository**: github.com/flipt-io/reverst

Reverst is a reverse-tunnel server built by Flipt, designed for their Hybrid Cloud product.

**Architecture**:
- Built on QUIC and HTTP/3
- Client opens a QUIC connection to the Reverst server, starts an initial stream, and writes a `RegisterListenerRequest`
- **Tunnel groups**: A load-balanced set of client-servers exposed through the Reverst HTTP interface. Multiple clients can register for the same tunnel group, providing redundancy and load balancing.
- QUIC provides persistent, bidirectional streaming connections over UDP

**Why they chose QUIC**: "QUIC offers secure (TLSv3 required) and persistent connections with bidirectional streaming, delivering the benefits of TCP without the HOL blocking problem, allowing Reverst to build persistent connections that handle multiple concurrent requests over unreliable networks."

**Relevance**: Reverst demonstrates using QUIC's stream mechanism to build a registration/control protocol, with load-balanced tunnel groups -- an architecture pattern relevant for a multi-tenant tunneling service.

### Specter (Go, QUIC + DHT)

**Repository**: github.com/zllovesuki/specter

**Architecture**:
- Uses QUIC exclusively for inter-node and node-client transport
- Incorporates DHT (Distributed Hash Table) for peer discovery and routing
- Supports L7 (HTTP/S) and L4 (TCP) traffic tunneling over a single TLS port
- Manages Let's Encrypt certificates via dns01 challenges

**Key design decisions**:
- **QUIC-only transport**: The transport layer uses only QUIC, though the interface is extensible
- **Redundant connections**: Client maintains connections to multiple edge nodes with self-healing to maintain the tunnel, avoiding single-point-of-failure issues
- **Distributed**: Uses DHT for service discovery, allowing a mesh of relay nodes

**Relevance**: Specter demonstrates a more ambitious architecture where QUIC enables a distributed, self-healing tunnel mesh. The DHT approach for multi-node discovery is interesting for scaling beyond a single relay server.

---

## 4. QUIC Multiplexing vs Yamux

### Is Yamux Needed with QUIC?

**No.** When using QUIC as the transport, yamux (or any other userspace multiplexer) is redundant and counter-productive. QUIC provides native stream multiplexing at the transport layer, with per-stream flow control and independent loss recovery.

Evidence from real implementations:
- **libp2p**: For TCP transports, libp2p layers yamux on top. For QUIC transports, libp2p uses QUIC's native streams directly, with no yamux layer.
- **rstun**: Uses QUIC streams directly for each tunneled connection, no yamux.
- **frp**: Replaces its TCP multiplexing with QUIC's native multiplexing when in QUIC mode.
- **Reverst**: Uses QUIC streams directly.

### Performance Comparison

Available evidence consistently favors QUIC native streams over yamux:

- **Connection saturation**: "Compared to the Rust Yamux implementation used in TCP stacks, QUIC implementation can better saturate a connection." (Sigma Prime QUIC networking blog)
- **Latency**: "A tunnel using Yamux over TLS multiplies the initial connection latency, but using QUIC for a tunnel instead could cut the painful extra latency in half." Yamux over TLS requires: TCP handshake (1 RTT) + TLS handshake (1-2 RTT) + yamux stream open. QUIC requires: QUIC handshake (1 RTT) + stream open (essentially free, stream creation is part of the QUIC frame).
- **HOL blocking**: Yamux over TCP still suffers from TCP-level HOL blocking. If a TCP segment is lost, all yamux streams are blocked. QUIC streams have independent loss recovery.

### When to Use Yamux on Top of QUIC

There is essentially no good reason to layer yamux on top of QUIC. Doing so would:
- Add unnecessary protocol overhead (yamux framing on top of QUIC framing)
- Negate QUIC's per-stream HOL blocking elimination (yamux would reintroduce ordered delivery within its own framing)
- Add latency for stream establishment
- Increase code complexity with no benefit

**The only valid scenario** for yamux remains TCP-based fallback connections. When the tunnel falls back to TCP+TLS (because UDP is blocked), yamux is needed to provide multiplexing over the single TCP connection. This means the tunnel codebase will need both:
- QUIC native streams (for the primary QUIC transport)
- Yamux over TCP+TLS (for the fallback transport)

The tunnel protocol above the transport layer should abstract over both, presenting a unified stream interface.

---

## 5. Practical Concerns

### UDP Port Requirements for Deployment

QUIC runs over UDP, typically on port 443 (to share the port number with HTTPS, aiding firewall traversal). Deployment requirements:

- The relay server must have a UDP port accessible from the internet (usually 443)
- Many firewalls allow UDP/443 because it is used by HTTP/3/QUIC for normal web browsing
- If using a non-standard port, UDP firewall rules must explicitly allow it
- Unlike TCP, there is no `listen()` backlog concept -- the endpoint handles all incoming connections on the single UDP socket

### Cloud Provider Support

| Provider | UDP Support | QUIC-Specific Support |
|----------|-------------|----------------------|
| **AWS** | NLB supports UDP listeners (since 2019). QUIC passthrough mode added November 2025. UDP support for PrivateLink and dual-stack NLBs added October 2024. TCP_QUIC listener type allows both QUIC (UDP 443) and HTTPS fallback (TCP 443) on one listener. Available in all commercial and GovCloud regions at no additional cost. | Full QUIC support |
| **GCP** | External Application Load Balancer supports QUIC negotiation. UDP supported via external passthrough Network Load Balancers. GCP was the first major cloud to offer QUIC support for HTTPS load balancing. | Full QUIC support |
| **DigitalOcean** | Droplets support UDP natively (no firewall restrictions on outbound; inbound configurable via DO Firewall). However, DO Load Balancers historically support only TCP/HTTP/HTTPS -- no native UDP or QUIC load balancing as of the search results reviewed. | Limited -- no LB support found |
| **General** | Any VPS/bare-metal server supports UDP. The constraint is load balancer support if you need L4 load balancing for your relay servers. Direct-to-server UDP is universally supported. | Varies by provider |

**Recommendation**: For initial deployment, bind the QUIC endpoint directly on the relay server without a cloud load balancer. AWS NLB with QUIC passthrough is the best option for scaling with load balancing.

### Fallback Strategy: QUIC Primary, WebSocket/TCP Fallback

A robust tunneling product must handle environments where UDP is blocked. The recommended strategy mirrors what Chrome/Google does:

**Race-based connection establishment**:
1. Attempt QUIC (UDP) and WebSocket/TCP connections simultaneously
2. Use whichever establishes first
3. If QUIC connects first (most cases), cancel the TCP attempt
4. If TCP connects first (QUIC blocked), use TCP+TLS with yamux for multiplexing
5. Cache the result per network -- if QUIC failed once on this network, prefer TCP on subsequent connections (with periodic QUIC re-probes)

**Fallback transport options**:

| Fallback | Pros | Cons |
|----------|------|------|
| **WebSocket over HTTPS** | Traverses all firewalls and proxies, including corporate HTTP proxies. Uses port 443/TCP. Most universally compatible. | Adds framing overhead. Requires HTTP upgrade handshake. No native multiplexing (need yamux). |
| **Raw TCP + TLS** | Lower overhead than WebSocket. Simple implementation. | Cannot traverse HTTP proxies. Blocked by some firewalls on non-standard ports. |
| **HTTP/2 with CONNECT** | Good proxy traversal. Built-in multiplexing via HTTP/2 streams. | More complex protocol. Higher overhead than raw TCP. |

**Recommended approach**: QUIC primary + WebSocket-over-HTTPS fallback. WebSocket provides the broadest compatibility for restricted networks.

**Implementation architecture**:
```
                    +------------------+
                    | Tunnel Protocol  |  (auth, stream mapping, control messages)
                    +------------------+
                    | Transport Trait  |  (abstract stream interface)
                    +--------+---------+
                             |
               +-------------+-------------+
               |                           |
    +----------+----------+     +----------+----------+
    | QUIC Transport      |     | WS/TCP Transport    |
    | (Quinn streams)     |     | (WebSocket + yamux) |
    +---------------------+     +---------------------+
```

### UDP Blocking Prevalence

Verified data points on UDP blocking:
- **General internet**: Google's measurements indicate only a small percentage of connections fail over UDP. Chrome successfully uses QUIC for over half of all connections to Google servers.
- **Enterprise networks**: Many enterprise firewalls explicitly block QUIC/UDP. Security vendors (Palo Alto, Check Point, Zscaler) provide documented procedures for blocking QUIC. The motivation is that QUIC prevents SSL forward proxy inspection.
- **China (GFW)**: As of April 2024, China's Great Firewall began blocking QUIC connections to specific domains via SNI inspection. An average of 43.8K FQDNs from the Tranco list are blocked weekly. This uses deep packet inspection of QUIC Initial packets.
- **No precise global percentage**: There is no authoritative study giving an exact percentage of networks that block UDP globally. Google's characterization of "a small number" and Chrome's successful QUIC adoption suggests <5% of general internet connections, but enterprise environments have significantly higher blocking rates.

---

## 6. Recommendations

Based on this research, the following recommendations apply to building a QUIC-based tunnel in Rust:

### Transport Layer

1. **Use Quinn as the primary QUIC library**. It is the most mature, most downloaded, and best-documented QUIC implementation in Rust. Version 0.11.x is stable and production-ready.

2. **Use QUIC native streams for multiplexing**. Do not layer yamux on top of QUIC. Map each tunneled connection to a dedicated QUIC bidirectional stream.

3. **Implement WebSocket+TLS+yamux fallback**. Essential for enterprise environments where UDP is blocked. Abstract the transport behind a trait so the tunnel protocol is transport-agnostic.

4. **Use race-based transport selection**. Attempt QUIC and WebSocket simultaneously; use whichever connects first. Cache results per network.

### QUIC Configuration

5. **Enable 0-RTT for reconnections** but only for idempotent operations (stream opens, keepalives). Do not send authentication credentials via 0-RTT due to replay attack vulnerability.

6. **Enable connection migration** on the server. This allows tunnel agents to survive network changes (Wi-Fi to cellular, NAT rebinding) without reconnecting.

7. **Consider rstun's hop-interval pattern** for anti-throttling: periodically migrate to a new local UDP port to avoid ISP/carrier UDP throttling.

8. **Default to ChaCha20-Poly1305** cipher for broad device compatibility (no AES-NI required), with AES-256-GCM as alternative for hardware-accelerated environments.

### Deployment

9. **Bind QUIC on UDP/443** to maximize firewall traversal (same port as HTTP/3).

10. **Deploy relay servers on AWS** for best QUIC load balancing support (NLB with QUIC passthrough). GCP is the alternative. Avoid DigitalOcean load balancers for QUIC workloads.

11. **Budget for higher CPU usage** on relay servers compared to TCP. QUIC's userspace processing has measurable overhead. Consider GSO (Generic Segmentation Offload) and larger segment sizes to improve throughput.

### Architecture Pattern

12. **Follow the frp/Reverst pattern**: Offer QUIC as a transport option with the same tunnel protocol running over either QUIC or TCP+WS. This provides maximum flexibility.

13. **Use QUIC datagrams** (RFC 9221, supported by Quinn) for UDP tunnel traffic that does not need reliability, avoiding the overhead of QUIC's reliability mechanisms for inherently unreliable protocols.

---

## 7. References

All URLs listed below were fetched or consulted during this research.

### Quinn Library
- Quinn GitHub Repository: https://github.com/quinn-rs/quinn
- Quinn API Documentation (docs.rs): https://docs.rs/quinn/latest/quinn/
- Quinn Connection API: https://docs.rs/quinn/latest/quinn/struct.Connection.html
- Quinn SendStream API: https://docs.rs/quinn/latest/quinn/struct.SendStream.html
- Quinn RecvStream API: https://docs.rs/quinn/latest/quinn/struct.RecvStream.html
- Quinn Endpoint API: https://docs.rs/quinn/latest/quinn/struct.Endpoint.html
- Quinn Connecting (0-RTT) API: https://docs.rs/quinn/latest/quinn/struct.Connecting.html
- Quinn on crates.io: https://crates.io/crates/quinn
- Quinn crates.io versions: https://crates.io/crates/quinn/versions
- Quinn DeepWiki: https://deepwiki.com/quinn-rs/quinn
- Quinn Data Transfer Guide: https://quinn-rs.github.io/quinn/quinn/data-transfer.html

### QUIC Protocol and Performance
- QUIC Wikipedia: https://en.wikipedia.org/wiki/QUIC
- Cloudflare -- The Road to QUIC: https://blog.cloudflare.com/the-road-to-quic/
- Catchpoint -- QUIC vs TCP: https://www.catchpoint.com/http2-vs-http3/quic-vs-tcp
- APNIC -- Comparing TCP and QUIC: https://blog.apnic.net/2022/11/03/comparing-tcp-and-quic/
- APNIC -- A QUIC progress report (2025): https://blog.apnic.net/2025/06/17/a-quic-progress-report/
- Fastly -- Measuring QUIC vs TCP Computational Efficiency: https://www.fastly.com/blog/measuring-quic-vs-tcp-computational-efficiency
- Codavel -- QUIC vs TCP+TLS: https://medium.com/codavel-blog/quic-vs-tcp-tls-and-why-quic-is-not-the-next-big-thing-d4ef59143efd
- ETH Zurich -- Continuous benchmarking of open-source QUIC implementations (2024): https://nsg.ee.ethz.ch/files/public/theses/2024-quic_benchmark_suite/thesis-1.pdf
- QUIC is not Quick Enough over Fast Internet: https://arxiv.org/html/2310.09423v2
- QUIC on the Fast Lane (ScienceDirect): https://www.sciencedirect.com/science/article/pii/S014036642400166X
- SIGCOMM 2020 -- QUIC CPU Performance: https://conferences.sigcomm.org/sigcomm/2020/files/slides/epiq/0%20QUIC%20and%20HTTP_3%20CPU%20Performance.pdf

### Research Papers
- Implementation and Performance Evaluation of TCP over QUIC Tunnels (arXiv:2504.10054): https://arxiv.org/abs/2504.10054
- Performance Evaluation of Various QUIC Implementations (DiVA): https://www.diva-portal.org/smash/get/diva2:1691838/FULLTEXT03
- Revisiting QUIC attacks (Springer): https://link.springer.com/article/10.1007/s10207-022-00630-6
- 0-RTT Attack and Defense of QUIC Protocol (IEEE): https://ieeexplore.ieee.org/document/9024637/
- TLS 1.3 -- Using TLS to Secure QUIC (RFC 9001): https://www.rfc-editor.org/rfc/rfc9001.html
- Accelerating QUIC with AF_XDP: https://jhc.sjtu.edu.cn/~shizhenzhao/ica3pp2023.pdf
- In-kernel QUIC implementation: https://github.com/lxin/quic

### Tunnel Implementations
- rstun (Rust, QUIC tunnel): https://github.com/neevek/rstun
- frp (Go, reverse proxy with QUIC): https://github.com/fatedier/frp
- frp QUIC network documentation: https://gofrp.org/en/docs/features/common/network/network/
- Reverst (Go, QUIC reverse tunnel by Flipt): https://github.com/flipt-io/reverst
- Reverst blog post: https://blog.flipt.io/so-we-built-reverst
- Specter (Go, QUIC + DHT tunnel): https://github.com/zllovesuki/specter
- awesome-tunneling list: https://github.com/anderspitman/awesome-tunneling

### Multiplexing
- yamux (Rust): https://github.com/libp2p/rust-yamux
- libp2p yamux documentation: https://docs.libp2p.io/concepts/multiplex/yamux/
- libp2p stream multiplexing overview: https://docs.libp2p.io/concepts/multiplex/overview/
- Sigma Prime -- QUIC networking and multiplexing: https://blog.sigmaprime.io/quic-networking.html

### Firewall and Blocking
- USENIX Security 2025 -- QUIC Censorship by Great Firewall of China: https://gfw.report/publications/usenixsecurity25/en/
- Palo Alto Networks -- Block QUIC protocol: https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClarCAC
- Zscaler -- Managing the QUIC Protocol: https://help.zscaler.com/zia/managing-quic-protocol
- Check Point -- QUIC protocol discussion: https://community.checkpoint.com/t5/Security-Gateways/Quic-protocol/td-p/158382
- IETF -- Applicability of the QUIC Transport Protocol: https://quicwg.org/ops-drafts/draft-ietf-quic-applicability.html

### Cloud Provider Support
- AWS -- QUIC Protocol Support for NLB: https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-quic-protocol-support-for-network-load-balancer-accelerating-mobile-first-applications/
- AWS -- NLB QUIC passthrough mode (Nov 2025): https://aws.amazon.com/about-aws/whats-new/2025/11/aws-network-load-balancer-quic-passthrough-mode/
- AWS -- UDP support for PrivateLink (Oct 2024): https://aws.amazon.com/about-aws/whats-new/2024/10/aws-udp-privatelink-dual-stack-network-load-balancers/
- AWS -- UDP Load Balancing for NLB: https://aws.amazon.com/blogs/aws/new-udp-load-balancing-for-network-load-balancer/
- GCP -- QUIC support for HTTPS load balancing: https://cloud.google.com/blog/products/gcp/introducing-quic-support-https-load-balancing
- GCP -- UDP with external passthrough NLB: https://cloud.google.com/load-balancing/docs/network/udp-with-network-load-balancing

### 0-RTT
- Iroh -- 0-RTT API blog post: https://www.iroh.computer/blog/0rtt-api
- Chromium -- QUIC fallback to TCP: https://groups.google.com/a/chromium.org/g/proto-quic/c/zo7--OQLQBo
