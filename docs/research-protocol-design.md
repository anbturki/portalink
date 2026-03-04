# Tunnel Protocol Design Patterns: Research Report

## Executive Summary

This document analyzes the protocol design patterns of four production tunnel implementations: **bore**, **rathole**, **frp**, and **chisel**. Each takes a distinct approach to the core challenges of tunnel software: connection lifecycle management, control/data channel separation, authentication, multiplexing, and error recovery. The findings are synthesized into a recommended protocol design for building a new tunnel in Rust.

---

## 1. bore (Rust, ~400 lines)

**Repository:** https://github.com/ekzhang/bore

### 1.1 Architecture Overview

bore is intentionally minimalist. It exposes a local TCP port to a remote server using a simple JSON-over-TCP protocol. The entire codebase is approximately 400 lines of async Rust using tokio.

**Key dependencies** (from [Cargo.toml](https://github.com/ekzhang/bore/blob/main/Cargo.toml)):
- `tokio` with full features (async runtime)
- `tokio-util` 0.7.1 with `codec` feature (for `LengthDelimitedCodec` framing)
- `serde` 1.0.136 with `derive` (serialization)
- `serde_json` 1.0.79 (JSON encoding)
- `hmac` 0.12.1 (authentication)
- `sha2` 0.10.2 (hashing)
- `uuid` 1.2.1 with `serde` and `v4` features (connection pairing)

### 1.2 Control Message Format

Messages are serialized as JSON and framed using tokio-util's `LengthDelimitedCodec`, which prepends a length prefix to each message. This gives a wire format of:

```
[4 bytes: length][N bytes: JSON payload]
```

Two enums define the full protocol vocabulary (from [shared.rs](https://github.com/ekzhang/bore/blob/main/src/shared.rs)):

**ClientMessage** (client -> server):
- `Authenticate(String)` -- HMAC response to server's authentication challenge
- `Hello(u16)` -- Initial message specifying desired port to forward
- `Accept(Uuid)` -- Accepts an incoming TCP connection, using the stream as proxy

**ServerMessage** (server -> client):
- `Challenge(Uuid)` -- Authentication challenge (sent first if auth enabled)
- `Hello(u16)` -- Response with actual assigned public port
- `Heartbeat` -- No-op to test client reachability
- `Connection(Uuid)` -- Requests client to accept a forwarded TCP connection
- `Error(String)` -- Server error, terminates the connection

### 1.3 Connection Lifecycle

```
Client                                    Server
  |                                         |
  |--- TCP connect to control port -------->|
  |                                         |
  |<-------- Challenge(uuid) ---------------|  (if auth enabled)
  |--- Authenticate(hmac_response) -------->|
  |                                         |  (server validates HMAC)
  |--- Hello(desired_port) ---------------->|
  |<-------- Hello(actual_port) ------------|
  |                                         |
  |          [tunnel is now active]         |
  |                                         |
  |<-------- Connection(uuid) --------------|  (new visitor arrives)
  |                                         |
  |--- new TCP to control port ------------>|  (separate TCP stream)
  |--- Accept(uuid) ---------------------->|
  |                                         |  (server pairs streams)
  |<========= bidirectional proxy =========>|
  |                                         |
  |<-------- Heartbeat --------------------|  (periodic keepalive)
```

### 1.4 Connection Pairing (UUID-based)

When a visitor connects to the public port, the server:
1. Generates a random UUID for that connection
2. Sends `Connection(uuid)` to the client on the control channel
3. Stores the visitor's TCP stream in a `HashMap<Uuid, TcpStream>`
4. Waits for the client to connect on a **new, separate TCP stream** and send `Accept(uuid)`
5. Looks up the UUID in the HashMap, extracts the visitor stream
6. Proxies bidirectionally between the two TCP streams

**Important design detail:** Incoming connections are stored for a maximum of **10 seconds** before being discarded, preventing memory leaks if a client fails to accept.

### 1.5 Authentication (HMAC-SHA256 Challenge-Response)

Authentication is implemented in [auth.rs](https://github.com/ekzhang/bore/blob/main/src/auth.rs):

1. An `Authenticator` struct wraps `Hmac<Sha256>`
2. The HMAC key is derived by SHA256-hashing the shared secret
3. Server sends a `Challenge(uuid)` with a randomly generated UUID as the challenge nonce
4. Client computes `HMAC-SHA256(key, uuid)` and returns the hex-encoded result via `Authenticate(response)`
5. Server verifies using constant-time comparison (`verify_slice`)
6. Authentication is required on **every TCP connection** (both control and data), not just the initial handshake

### 1.6 Error Handling & Reconnection

bore does **not** implement automatic reconnection. If the control channel drops, the client exits. This is a deliberate design choice for simplicity. The heartbeat mechanism exists to detect dead connections early.

### 1.7 Key Design Insights from bore

- **Simplicity through separate TCP streams:** Rather than multiplexing, each proxied connection gets its own TCP stream paired by UUID
- **JSON over length-delimited framing:** Simple to debug, easy to extend
- **Per-connection authentication:** Every TCP stream is authenticated, preventing unauthorized data channel usage
- **Timeout-based cleanup:** 10-second TTL on pending connections prevents resource exhaustion

---

## 2. rathole (Rust)

**Repository:** https://github.com/rathole-org/rathole

### 2.1 Architecture Overview

rathole is a more feature-rich tunnel than bore, supporting multiple services, pluggable transports (TCP, TLS, Noise Protocol, WebSocket), hot-reload of configuration, and both TCP and UDP forwarding.

**Key dependencies** (from [Cargo.toml](https://github.com/rathole-org/rathole/blob/main/Cargo.toml)):
- `tokio` 1 with `full` features
- `serde` 1.0 with `derive`
- `bincode` 1 (binary serialization)
- `sha2` 0.10 (digest for service tokens)
- `backoff` 0.4 with `tokio` features (exponential backoff retry)

### 2.2 Control Message Format

rathole uses **bincode** for serialization -- a compact binary format with no metadata overhead. The protocol defines messages in [protocol.rs](https://github.com/rathole-org/rathole/blob/main/src/protocol.rs):

**Hello enum** (initial handshake on any connection):
- `ControlChannelHello(ProtocolVersion, Digest)` -- SHA256 sum of service name (acts as service identifier and auth token)
- `DataChannelHello(ProtocolVersion, Digest)` -- Token provided by `CreateDataChannel` command

**ControlChannelCmd enum** (commands on control channel):
- `CreateDataChannel` -- Server tells client to open a new data channel
- `HeartBeat` -- Keepalive

**DataChannelCmd enum** (commands on data channel):
- `StartForwardTcp` -- Begin TCP forwarding on this data channel
- `StartForwardUdp` -- Begin UDP forwarding on this data channel

The wire format uses bincode with `serialized_size` for pre-calculating packet sizes, giving a compact binary framing.

### 2.3 Control vs Data Channel Separation

**Control Channel:**
- Persistent long-lived connection from client to server
- Used exclusively for commands and heartbeats
- One control channel per service
- Server sends `CreateDataChannel` when a visitor arrives

**Data Channel:**
- Ephemeral connection created on-demand
- Client opens a new TCP connection (or stream) to server
- Sends `DataChannelHello` with the token received in `CreateDataChannel`
- After handshake, receives `StartForwardTcp` or `StartForwardUdp`
- Then enters raw proxy mode

### 2.4 Connection Lifecycle

```
Client                                      Server
  |                                           |
  |--- TCP connect --------------------------->|
  |--- ControlChannelHello(version, digest) -->|
  |                                           |  (server validates version + digest)
  |                                           |  digest = SHA256(service_name + token)
  |                                           |
  |      [control channel established]        |
  |                                           |
  |<--- CreateDataChannel(session_token) -----|  (visitor arrives)
  |                                           |
  |--- new TCP connection ------------------->|
  |--- DataChannelHello(version, token) ----->|
  |                                           |  (server validates token)
  |<--- StartForwardTcp ---------------------|
  |                                           |
  |<========= bidirectional proxy ===========>|
  |                                           |
  |<--- HeartBeat ----------------------------|  (periodic)
  |--- HeartBeat ---------------------------->|  (response)
```

### 2.5 Authentication

Authentication is **per-service** using mandatory tokens:
- Each service in the TOML config has a `token` field
- The `Digest` in `ControlChannelHello` is `SHA256(service_name || token)`
- The server validates this digest against its own computed value
- Data channels use session tokens (nonces) generated by the server when `CreateDataChannel` is sent, preventing replay attacks

### 2.6 Hot Reload

rathole supports dynamic addition/removal of services by watching the TOML configuration file:
- File system change events trigger config re-parsing
- New services spin up new control channels
- Removed services tear down their control channels
- No server restart required

### 2.7 Pluggable Transport Layer

Transport options are configured via TOML (`[client.transport]` / `[server.transport]`):
- **TCP:** Plain TCP (default)
- **TLS:** Certificate-based with PKCS#12 and PEM support
- **Noise Protocol:** Uses `Noise_NK_25519_ChaChaPoly_BLAKE2s` -- lightweight, easy-to-configure encryption. Requires only X25519 keypair (generated via `rathole --genkey`)
- **WebSocket:** HTTP-upgradeable transport, optionally with TLS

### 2.8 Error Handling & Reconnection

rathole uses the `backoff` crate for **exponential backoff retry**:
- When the control channel fails, the client retries with increasing delays
- Log messages show patterns like "Retry in 1s", "Retry in 2s", etc.
- Heartbeat timeout is configurable (default 40 seconds), must exceed `server.heartbeat_interval`
- Data channel failures do **not** automatically trigger reconnection (the control channel handles recovery by requesting new data channels)

### 2.9 Key Design Insights from rathole

- **Bincode over JSON:** More compact, faster serialization for a performance-focused tunnel
- **Session tokens for data channels:** Prevents unauthorized data channel creation
- **Pluggable transport:** Clean abstraction allowing multiple encryption/transport strategies
- **Service-oriented design:** Multiple tunnels per instance, each with independent auth
- **Exponential backoff:** Production-grade reconnection strategy

---

## 3. frp (Go)

**Repository:** https://github.com/fatedier/frp

### 3.1 Architecture Overview

frp (Fast Reverse Proxy) is the most feature-rich tunnel in this survey. It supports TCP, UDP, HTTP, HTTPS, STCP (secret TCP), XTCP (P2P), and SUDP proxy types. It uses JSON for message encoding and supports TCP stream multiplexing since v0.10.0.

### 3.2 Control Message Format

All messages implement the `msg.Message` interface defined in [`pkg/msg/msg.go`](https://github.com/fatedier/frp/tree/master/pkg/msg). Messages are serialized as JSON. The framing format uses a type byte + length-prefixed JSON:

```
[1 byte: message type][8 bytes: length][N bytes: JSON payload]
```

**Message types (from TypeMap registration):**

| Message Type        | Direction        | Purpose                                        |
|---------------------|------------------|-------------------------------------------------|
| `Login`             | Client -> Server | Authentication with token, version, OS, arch    |
| `LoginResp`         | Server -> Client | Response with version, run_id, error             |
| `NewProxy`          | Client -> Server | Register a proxy (type, name, config)            |
| `NewProxyResp`      | Server -> Client | Proxy registration result                        |
| `CloseProxy`        | Client -> Server | Deregister a proxy                               |
| `ReqWorkConn`       | Server -> Client | Request client to open a work connection         |
| `NewWorkConn`       | Client -> Server | Client registers a new work connection           |
| `StartWorkConn`     | Server -> Client | Server assigns work to a work connection         |
| `Ping`              | Client -> Server | Heartbeat ping                                   |
| `Pong`              | Server -> Client | Heartbeat response                               |
| `NewVisitorConn`    | Client -> Server | For STCP/XTCP visitor connections                |
| `NewVisitorConnResp`| Server -> Client | Response to visitor connection                   |

### 3.3 Connection Lifecycle

```
Client (frpc)                              Server (frps)
  |                                          |
  |--- TCP/KCP/QUIC/WebSocket connect ------>|
  |--- Login(token, version, os, arch) ----->|
  |<--- LoginResp(version, run_id, err) -----|
  |                                          |
  |--- NewProxy(name, type, config) -------->|  (for each proxy)
  |<--- NewProxyResp(remote_addr, err) ------|
  |                                          |
  |      [tunnel is now active]              |
  |                                          |
  |<--- ReqWorkConn -------------------------|  (visitor arrives)
  |                                          |
  |--- NewWorkConn(run_id, ...) ------------>|  (new connection)
  |<--- StartWorkConn(proxy_name, ...) ------|  (assign work)
  |                                          |
  |<========= bidirectional proxy ==========>|
  |                                          |
  |--- Ping -------------------------------->|  (periodic)
  |<--- Pong --------------------------------|
```

### 3.4 TCP Stream Multiplexing

Since v0.10.0, frp supports multiplexing all logical connections over a single TCP connection (configurable via `tcpMux`):
- Similar to HTTP/2 multiplexing
- All work connections for a single frpc are multiplexed through one TCP connection
- Reduces connection overhead and latency
- When `tcpMux` is enabled, the application-layer heartbeat defaults to disabled since the multiplexer has its own keepalive mechanism

### 3.5 HTTP Subdomain Routing

frp provides HTTP-aware routing for web tunnels:

1. **Subdomain routing:** Server config sets `subDomainHost` (e.g., `frps.example.com`). Client sets `subdomain: myapp`, routing `myapp.frps.example.com` to the local service
2. **Custom domain routing:** Clients can specify `customDomains` for their proxy
3. **Host header rewriting:** The `host_header_rewrite` option rewrites the `Host` header before forwarding to the local service (needed for apps that route based on Host)
4. **URL path routing:** `locations` field specifies URL path prefixes for routing
5. **HTTP CONNECT multiplexing:** `tcpmux` with `multiplexer = "httpconnect"` uses the Host in HTTP CONNECT requests for routing

### 3.6 Work Connection Pool

frp maintains a **pre-established pool of work connections** for faster response times:
- Server sends `ReqWorkConn` proactively to build up the pool
- When a visitor arrives, an available pooled connection is immediately assigned via `StartWorkConn`
- Reduces latency by avoiding connection setup on the critical path

### 3.7 Authentication

frp supports multiple authentication methods:
- **Token-based:** Simple shared secret in the `Login` message
- **OIDC (OpenID Connect):** For enterprise environments
- Authentication can be required on work connections too (`authenticate_new_work_conns`)
- The `Setter` interface allows pluggable auth on `Login`, `Ping`, and `NewWorkConn` messages

### 3.8 Key Design Insights from frp

- **Rich message vocabulary:** Dedicated message types for every protocol action
- **Work connection pooling:** Pre-established connections reduce per-request latency
- **TCP multiplexing:** Optional single-connection mode reduces overhead
- **HTTP-aware routing:** Host header inspection, subdomain routing, URL path routing
- **Pluggable auth:** Token and OIDC support through interface abstraction

---

## 4. chisel (Go)

**Repository:** https://github.com/jpillora/chisel

### 4.1 Architecture Overview

chisel takes a fundamentally different approach: it tunnels TCP/UDP over **WebSocket + SSH**. This enables traversal of HTTP-only firewalls while getting robust multiplexing and encryption from the SSH protocol. Single binary serves as both client and server.

### 4.2 WebSocket Transport Layer

The connection lifecycle begins with an HTTP upgrade:

1. Client initiates HTTP connection to server
2. HTTP upgrades to WebSocket (using `gorilla/websocket`)
3. SSH handshake runs **over the WebSocket connection** (using `golang.org/x/crypto/ssh`)
4. All tunnel traffic flows as SSH channels within this single WebSocket/TCP connection

This layered approach means:
- **WebSocket** provides HTTP compatibility (firewall traversal)
- **SSH** provides encryption (ECDSA with SHA256), authentication, and multiplexing
- **SSH channels** provide logical stream multiplexing

### 4.3 Multiplexing Architecture

Chisel's multiplexing is built on SSH channels:

```
[TCP Connection]
  -> [WebSocket]
    -> [SSH Session]
      -> [SSH Channel 1: proxy stream A]
      -> [SSH Channel 2: proxy stream B]
      -> [SSH Channel 3: proxy stream C]
      -> ...
```

Key characteristics:
- One TCP connection per client (all traffic multiplexed)
- Each remote (port forward) maps 1:1 to a proxy
- Proxies listen on local/remote ports and forward data through SSH channels
- Both forward and reverse port forwarding supported
- UDP multiplexing requires mapping source ports to channels in memory

The `Tunnel` struct (in [tunnel.go](https://github.com/jpillora/chisel/blob/master/share/tunnel/tunnel.go)) represents the SSH tunnel. Both client and server are `Tunnel` instances:
- Client has a single set of remotes
- Server has multiple sets (one per connected client)

### 4.4 Connection Lifecycle

```
Client                                      Server
  |                                           |
  |--- HTTP GET /upgrade ------------------->|
  |<--- 101 Switching Protocols (WebSocket) -|
  |                                           |
  |--- SSH Handshake (over WebSocket) ------>|
  |    - Key exchange (ECDSA)                 |
  |    - Server fingerprint verification      |
  |    - Client auth (user:pass if set)       |
  |<--- SSH Handshake Complete --------------|
  |                                           |
  |    [SSH Session established]              |
  |                                           |
  |--- handleSSHRequests (goroutine) -------->|
  |--- handleSSHChannels (goroutine) -------->|
  |                                           |
  |--- SSH Channel Open (remote:port) ------>|  (for each tunnel)
  |<========= bidirectional proxy ===========>|
```

### 4.5 Authentication

Chisel uses **SSH's built-in authentication mechanisms:**

**Server Authentication:**
- Server generates an in-memory ECDSA public/private key pair on startup
- Server displays the public key fingerprint (SHA256)
- Client can verify server identity using `--fingerprint` flag
- Fingerprint mismatch closes the connection immediately

**Client Authentication:**
- Optional username/password via `--auth user:pass`
- Server validates against `--authfile` (a users configuration file)
- Uses SSH's Password authentication method internally

### 4.6 Error Handling & Reconnection

chisel includes built-in reconnection:
- Client automatically reconnects on connection loss
- Configurable retry behavior with backoff
- WebSocket provides clean close notifications
- SSH keepalive detects dead connections

### 4.7 Key Design Insights from chisel

- **Leveraging existing protocols:** SSH provides battle-tested encryption, auth, and multiplexing
- **WebSocket for firewall traversal:** HTTP-compatible transport passes through corporate firewalls
- **Zero custom protocol:** No custom message format needed -- SSH channels handle everything
- **Single binary:** Client and server in one executable simplifies distribution
- **Fingerprint-based trust:** Server identity verification without CA infrastructure

---

## 5. Comparative Analysis

### 5.1 Message Serialization

| Project  | Format      | Framing               | Pros                          | Cons                       |
|----------|-------------|------------------------|-------------------------------|----------------------------|
| bore     | JSON        | Length-delimited (4B)  | Human-readable, debuggable    | Verbose, slower parsing    |
| rathole  | bincode     | bincode built-in       | Compact, fast, zero-copy      | Not human-readable         |
| frp      | JSON        | Type(1B) + Length(8B)  | Debuggable, extensible        | Verbose                    |
| chisel   | SSH protocol| SSH framing            | Battle-tested, standard       | Tied to SSH semantics      |

### 5.2 Connection Multiplexing

| Project  | Strategy                    | Connections per client  |
|----------|-----------------------------|-------------------------|
| bore     | One TCP stream per proxy    | N + 1 (1 control + N data) |
| rathole  | One TCP stream per proxy    | N + 1 (1 control + N data) |
| frp      | Optional TCP mux (yamux)    | 1 (when tcpMux enabled) or N+1 |
| chisel   | SSH channel multiplexing    | 1 (always)              |

### 5.3 Authentication

| Project  | Mechanism                    | Per-connection | Notes                         |
|----------|------------------------------|----------------|-------------------------------|
| bore     | HMAC-SHA256 challenge-response| Yes           | Every TCP stream authenticated |
| rathole  | SHA256(service+token) digest | Control only   | Data channels use session tokens |
| frp      | Token / OIDC                 | Optional       | Configurable per message type |
| chisel   | SSH password + fingerprint   | Session-level  | Uses SSH built-in auth        |

### 5.4 Reconnection Strategy

| Project  | Auto-reconnect | Strategy                |
|----------|----------------|--------------------------|
| bore     | No             | Client exits on failure  |
| rathole  | Yes            | Exponential backoff (via `backoff` crate) |
| frp      | Yes            | Configurable retry       |
| chisel   | Yes            | Automatic with backoff   |

### 5.5 Transport Flexibility

| Project  | Transports supported                        |
|----------|---------------------------------------------|
| bore     | TCP only                                    |
| rathole  | TCP, TLS, Noise Protocol, WebSocket         |
| frp      | TCP, KCP, QUIC, WebSocket                   |
| chisel   | WebSocket (HTTP upgradeable)                |

---

## 6. Recommended Protocol Design for Our Tunnel

Based on the analysis above, the following design picks the best patterns from each project while maintaining Rust idiomacy and production readiness.

### 6.1 Overall Architecture

Adopt **rathole's dual-channel architecture** with **frp's rich message vocabulary**:

```
Control Channel: Long-lived, authenticated, carries commands only
Data Channels:   Ephemeral, token-validated, carry proxied traffic
```

### 6.2 Message Format

Use a **hybrid approach**:
- **Development/debug builds:** JSON with length-delimited framing (bore's approach) for easy debugging
- **Production:** bincode with length-delimited framing (rathole's approach) for performance

Recommended wire format:

```
[1 byte: message type tag][4 bytes: payload length (big-endian)][N bytes: payload]
```

The type tag enables quick dispatch without deserializing the payload. Combine bore's simplicity with frp's type system.

### 6.3 Message Types

```rust
/// Messages from client to server
#[derive(Serialize, Deserialize)]
enum ClientMessage {
    /// Initial authentication
    Auth {
        version: String,
        token_hash: Vec<u8>,     // HMAC-SHA256 of challenge
    },
    /// Register a tunnel
    RegisterTunnel {
        tunnel_id: String,
        tunnel_type: TunnelType, // TCP, HTTP, UDP
        config: TunnelConfig,    // port, subdomain, host_header_rewrite, etc.
    },
    /// Deregister a tunnel
    DeregisterTunnel {
        tunnel_id: String,
    },
    /// Accept a data channel assignment
    AcceptDataChannel {
        session_token: Uuid,
    },
    /// Heartbeat response
    Pong,
}

/// Messages from server to client
#[derive(Serialize, Deserialize)]
enum ServerMessage {
    /// Authentication challenge
    AuthChallenge {
        nonce: Uuid,
        server_version: String,
    },
    /// Authentication result
    AuthResult {
        success: bool,
        client_id: Uuid,
        error: Option<String>,
    },
    /// Tunnel registration result
    TunnelRegistered {
        tunnel_id: String,
        public_addr: String,     // e.g., "myapp.tunnel.example.com:443"
        error: Option<String>,
    },
    /// Request client to open a data channel
    OpenDataChannel {
        session_token: Uuid,
        tunnel_id: String,
    },
    /// Heartbeat
    Ping,
    /// Server error
    Error {
        code: ErrorCode,
        message: String,
    },
}

/// Tunnel types
#[derive(Serialize, Deserialize)]
enum TunnelType {
    Tcp { remote_port: Option<u16> },
    Http { subdomain: Option<String>, custom_domain: Option<String> },
    Udp { remote_port: u16 },
}
```

### 6.4 Connection Lifecycle

```
Phase 1: Authentication (on control channel)
  Client -> Server: TCP connect
  Server -> Client: AuthChallenge { nonce, server_version }
  Client -> Server: Auth { version, token_hash: HMAC(secret, nonce) }
  Server -> Client: AuthResult { success, client_id }

Phase 2: Tunnel Registration (on control channel)
  Client -> Server: RegisterTunnel { tunnel_id, type, config }
  Server -> Client: TunnelRegistered { tunnel_id, public_addr }
  (repeat for each tunnel)

Phase 3: Traffic Proxying (data channels)
  [Visitor connects to public_addr]
  Server -> Client: OpenDataChannel { session_token, tunnel_id }
  Client -> Server: new TCP connection + AcceptDataChannel { session_token }
  Server:          validates token, pairs connections
  [Bidirectional proxy begins]

Phase 4: Keepalive (on control channel)
  Server -> Client: Ping (every 15 seconds)
  Client -> Server: Pong (within 30 seconds or connection is dropped)

Phase 5: Shutdown
  Client -> Server: DeregisterTunnel { tunnel_id }
  OR: TCP connection closes -> server cleans up all tunnels for that client
```

### 6.5 Authentication Design

Adopt **bore's HMAC challenge-response** with improvements:

1. Server sends `AuthChallenge` with a random UUID nonce
2. Client computes `HMAC-SHA256(key=SHA256(shared_secret), message=nonce)`
3. Server verifies with constant-time comparison
4. Data channels use server-generated **session tokens** (from rathole's approach) rather than re-authenticating -- reduces latency

### 6.6 Data Channel Management

Adopt a **hybrid of bore's UUID pairing and rathole's session tokens**:

1. When a visitor arrives, server generates a `session_token` (UUID v4)
2. Server stores `session_token -> visitor_stream` in a `DashMap` with a **10-second TTL** (bore's approach)
3. Server sends `OpenDataChannel { session_token }` to client
4. Client opens a new TCP connection and sends `AcceptDataChannel { session_token }`
5. Server validates token, extracts visitor stream, begins proxying
6. For HTTP tunnels, server does **Host header inspection** (frp's approach) to route to the correct tunnel

### 6.7 Reconnection Strategy

Adopt **rathole's exponential backoff** using the `backoff` crate:

```rust
use backoff::ExponentialBackoff;

let backoff = ExponentialBackoff {
    initial_interval: Duration::from_secs(1),
    max_interval: Duration::from_secs(60),
    max_elapsed_time: None, // retry forever
    ..Default::default()
};
```

On control channel disconnect:
1. All active tunnels are marked stale
2. Client retries connection with exponential backoff
3. On reconnection, client re-registers all tunnels
4. Server detects stale tunnel registrations and cleans up

### 6.8 Transport Layer

Adopt **rathole's pluggable transport** design:

```rust
#[async_trait]
trait Transport: Send + Sync {
    type Stream: AsyncRead + AsyncWrite + Send + Unpin;
    async fn connect(&self, addr: &str) -> Result<Self::Stream>;
    async fn accept(&self) -> Result<Self::Stream>;
}

// Implementations:
// - TcpTransport (default, no encryption)
// - TlsTransport (rustls-based)
// - NoiseTransport (snow crate, Noise_NK_25519_ChaChaPoly_BLAKE2s)
// - WebSocketTransport (tokio-tungstenite, for firewall traversal)
```

### 6.9 HTTP Tunnel Routing (from frp)

For HTTP tunnels, implement Host-header-based routing:

1. Server listens on ports 80/443
2. Reads the `Host` header from incoming HTTP requests (or SNI from TLS ClientHello)
3. Routes to the correct tunnel based on subdomain/domain matching
4. Optionally rewrites the `Host` header before forwarding (frp's `host_header_rewrite`)

### 6.10 Future Considerations: Stream Multiplexing

For v2, consider adding **optional stream multiplexing** (frp's `tcpMux` approach):
- Use the `yamux` crate for Rust-native stream multiplexing
- All data channels multiplexed over a single TCP connection
- Reduces connection overhead in high-throughput scenarios
- Make it opt-in, keeping the simple per-connection model as default

---

## 7. Summary: Best Patterns by Source

| Pattern                            | Source Project | Why                                              |
|------------------------------------|----------------|--------------------------------------------------|
| JSON + length-delimited framing    | bore           | Simple, debuggable, good for v1                  |
| HMAC-SHA256 challenge-response     | bore           | Secure, simple, prevents replay attacks          |
| UUID-based connection pairing      | bore           | Clean, race-free connection matching             |
| 10-second TTL on pending conns     | bore           | Prevents memory leaks                            |
| Control/data channel separation    | rathole        | Clean architecture, independent lifecycle        |
| Session tokens for data channels   | rathole        | Secure data channel establishment                |
| Bincode serialization (production) | rathole        | Performance-optimized wire format                |
| Pluggable transport layer          | rathole        | Flexibility (TCP/TLS/Noise/WebSocket)            |
| Exponential backoff reconnection   | rathole        | Production-grade reliability                     |
| Hot-reload configuration           | rathole        | Zero-downtime service changes                    |
| Rich message type vocabulary       | frp            | Extensible protocol                              |
| Work connection pooling            | frp            | Reduced per-request latency                      |
| HTTP subdomain routing             | frp            | HTTP-aware tunnel routing                        |
| Host header rewriting              | frp            | Compatibility with virtual-host-aware apps       |
| TCP stream multiplexing (optional) | frp            | Reduced connection overhead at scale             |
| WebSocket transport for traversal  | chisel         | Passes through HTTP-only firewalls               |
| SSH-level encryption + auth        | chisel         | Battle-tested security primitives                |
| Single binary client+server        | chisel/bore    | Simple distribution                              |

---

## References

### bore
- Repository: https://github.com/ekzhang/bore
- `src/shared.rs` (ClientMessage, ServerMessage enums, JSON framing): https://github.com/ekzhang/bore/blob/main/src/shared.rs
- `src/server.rs` (connection handling, UUID pairing, 10s TTL): https://github.com/ekzhang/bore/blob/main/src/server.rs
- `src/client.rs` (client connection, handshake, proxy): https://github.com/ekzhang/bore/blob/main/src/client.rs
- `src/auth.rs` (HMAC-SHA256 challenge-response authentication): https://github.com/ekzhang/bore/blob/main/src/auth.rs
- `src/main.rs` (CLI entry point): https://github.com/ekzhang/bore/blob/main/src/main.rs
- `Cargo.toml` (dependencies: tokio-util, serde_json, hmac, sha2, uuid): https://github.com/ekzhang/bore/blob/main/Cargo.toml

### rathole
- Repository: https://github.com/rathole-org/rathole
- `src/protocol.rs` (Hello, ControlChannelCmd, DataChannelCmd, bincode serialization): https://github.com/rathole-org/rathole/blob/main/src/protocol.rs
- `src/server.rs` (control channel, data channel handling): https://github.com/rathole-org/rathole/blob/main/src/server.rs
- `src/client.rs` (reconnection with backoff, control channel lifecycle): https://github.com/rathole-org/rathole/blob/main/src/client.rs
- `docs/transport.md` (pluggable transport: TCP, TLS, Noise, WebSocket): https://github.com/rapiz1/rathole/blob/main/docs/transport.md
- `Cargo.toml` (dependencies: bincode, sha2, backoff, tokio): https://github.com/rathole-org/rathole/blob/main/Cargo.toml
- DeepWiki architecture overview: https://deepwiki.com/rathole-org/rathole

### frp
- Repository: https://github.com/fatedier/frp
- `pkg/msg/msg.go` (message types: Login, LoginResp, NewProxy, ReqWorkConn, etc.): https://github.com/fatedier/frp/tree/master/pkg/msg
- `client/control.go` (client control connection, message handling): https://github.com/fatedier/frp/blob/master/client/control.go
- `server/control.go` (server control, work connection management): https://github.com/fatedier/frp/blob/master/server/control.go
- `client/proxy/proxy.go` (proxy implementation, StartWorkConn handling): https://github.com/fatedier/frp/blob/master/client/proxy/proxy.go
- `pkg/auth` (Setter interface, token/OIDC auth): https://pkg.go.dev/github.com/fatedier/frp/pkg/auth
- DeepWiki architecture overview: https://deepwiki.com/fatedier/frp
- Official documentation: https://gofrp.org/en/

### chisel
- Repository: https://github.com/jpillora/chisel
- `share/tunnel/tunnel.go` (Tunnel struct, SSH channel handling): https://github.com/jpillora/chisel/blob/master/share/tunnel/tunnel.go
- `share/tunnel/tunnel_in_proxy.go` (TCP proxy over SSH channels): https://github.com/jpillora/chisel/blob/master/share/tunnel/tunnel_in_proxy.go
- `share/tunnel/tunnel_in_proxy_udp.go` (UDP multiplexing over SSH): https://github.com/jpillora/chisel/blob/master/share/tunnel/tunnel_in_proxy_udp.go
- `client/client.go` (client initialization, WebSocket connection): https://github.com/jpillora/chisel/blob/master/client/client.go
- `client/client_connect.go` (WebSocket dial, SSH over WebSocket): https://github.com/jpillora/chisel/blob/master/client/client_connect.go
- `server/server.go` (server setup, HTTP upgrade, multi-client management): https://github.com/jpillora/chisel/blob/master/server/server.go
- Go package docs: https://pkg.go.dev/github.com/jpillora/chisel/share/tunnel
- DeepWiki architecture overview: https://deepwiki.com/jpillora/chisel

### Supporting References
- tokio-util LengthDelimitedCodec: https://docs.rs/tokio-util/latest/tokio_util/codec/length_delimited/
- Yamux specification (stream multiplexing): https://github.com/hashicorp/yamux/blob/master/spec.md
- rathole crates.io dependencies: https://crates.io/crates/rathole/dependencies
