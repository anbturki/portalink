# Portalink Implementation Plan

Phased implementation with small, testable milestones. Each phase is independently verifiable.

---

## Phase 0: Project Scaffold

**Goal:** Cargo workspace with crate structure, dependencies, and CI basics.

### Tasks
1. Initialize Cargo workspace with 4 crates:
   - `portalink-core` — Protocol types, transport trait, frame codec, shared types
   - `portalink-server` — Server binary logic
   - `portalink-client` — Client binary logic
   - `portalink-cli` — CLI entry point (thin wrapper, uses clap)
2. Add all dependencies with pinned versions
3. Set Rust edition 2024, MSRV 1.88
4. Add `rustfmt.toml` and `clippy.toml` with strict settings
5. Add release profile optimizations (LTO, strip, panic=abort)
6. Create `.github/workflows/ci.yml` — build + test + clippy + fmt

### How to Test
```bash
cargo build --workspace          # All crates compile
cargo test --workspace           # No tests yet, but framework works
cargo clippy --workspace         # Zero warnings
cargo fmt --check                # Formatting correct
```

### Deliverable
A clean workspace that compiles with zero warnings.

---

## Phase 1: Protocol Types & Frame Codec

**Goal:** Define all message types and implement binary frame encoding/decoding.

### Tasks
1. Define `ClientMessage` and `ServerMessage` enums in `portalink-core`
2. Define `TunnelType` (Http, Tcp, Udp) and `TunnelConfig` structs
3. Implement frame codec:
   - Encode: `message → postcard bytes → [1B type][4B length][payload]`
   - Decode: Read type tag + length, deserialize payload
4. Add max frame size validation (64KB for control messages)
5. Unit tests for round-trip serialization of every message variant
6. Fuzz-style tests: invalid type tags, truncated frames, oversized frames

### How to Test
```bash
cargo test -p portalink-core     # All codec tests pass
```
**Manual validation:** Read the test output — every message type encodes and decodes correctly. Invalid inputs are rejected with proper errors.

### Security Checkpoint
- [ ] Oversized frames are rejected before allocation
- [ ] Invalid type tags produce errors, not panics
- [ ] No `unwrap()` in codec paths — all errors propagated

---

## Phase 2: Transport Trait & TCP Transport

**Goal:** Abstract transport layer with a working TCP implementation.

### Tasks
1. Define `Transport` trait in `portalink-core`
2. Implement `TcpTransport` (connect, bind, accept)
3. Add configurable connection timeouts
4. Add `FramedStream<S>` wrapper that reads/writes framed messages over any `AsyncRead + AsyncWrite`
5. Integration test: spawn TCP listener, connect, exchange framed messages

### How to Test
```bash
cargo test -p portalink-core -- --test-threads=1    # TCP tests need sequential ports
```
**Manual validation:** The integration test proves two ends can exchange typed messages over TCP.

### Security Checkpoint
- [ ] Connection timeout is enforced (default 10s)
- [ ] Socket options set: `TCP_NODELAY`, `SO_KEEPALIVE`

---

## Phase 3: Authentication & Control Channel

**Goal:** Client authenticates to server via HMAC-SHA256 challenge-response.

### Tasks
1. Server generates random nonce (UUID v4), sends `AuthChallenge`
2. Client computes `HMAC-SHA256(secret, nonce)`, sends `Auth`
3. Server verifies using `constant_time_eq`, sends `AuthResult`
4. Implement keepalive: server sends `Ping` every 15s, client replies `Pong`
5. Disconnect on: auth failure, missed pong (30s timeout)
6. Integration test: successful auth, failed auth, keepalive timeout

### How to Test
```bash
cargo test -p portalink-core     # Auth protocol tests
```
**Manual validation:**
1. Start server in one terminal
2. Connect with correct secret → see "authenticated" log
3. Connect with wrong secret → see "auth failed", connection drops
4. Connect and wait 45s → see keepalive timeout, connection drops

### Security Checkpoint
- [ ] Nonce is generated with `rand::thread_rng()` (CSPRNG)
- [ ] `constant_time_eq` used for hash comparison
- [ ] Secret is never logged, even at trace level
- [ ] Auth timeout: reject if auth not completed within 5s

---

## Phase 4: Tunnel Registration

**Goal:** Client registers tunnels, server tracks them.

### Tasks
1. Client sends `RegisterTunnel` with type + config after auth
2. Server validates tunnel config:
   - Subdomain: alphanumeric + hyphens, 3-63 chars, not reserved
   - TCP port: within allowed range
3. Server stores tunnel in `DashMap<TunnelId, TunnelInfo>`
4. Server responds with `TunnelRegistered { public_addr }`
5. Support multiple tunnels per client
6. Deregistration on client disconnect (cleanup)
7. Unit tests: valid/invalid subdomain names, duplicate registration

### How to Test
```bash
cargo test -p portalink-server   # Registration tests
```
**Manual validation:**
1. Start server
2. Client connects, authenticates, registers `http://test.localhost:8080`
3. Server logs: "Tunnel registered: test → 127.0.0.1:8080"
4. Client disconnects → server logs: "Tunnel deregistered: test"

### Security Checkpoint
- [ ] Subdomain validation prevents injection (no dots, no slashes)
- [ ] Reserved subdomains blocked (api, www, admin, etc.)
- [ ] Max tunnels per client enforced (default: 10)

---

## Phase 5: Data Channel & TCP Proxy

**Goal:** Visitors connect to public port, traffic proxies to local service.

### Tasks
1. Server opens TCP listener on tunnel's public port
2. When visitor connects:
   - Generate session token (UUID v4)
   - Store in `DashMap` with 10s TTL
   - Send `OpenDataChannel { session_token }` to client
3. Client receives `OpenDataChannel`:
   - Opens new TCP connection to server
   - Sends `AcceptDataChannel { session_token }`
4. Server validates token (constant-time comparison), pairs streams
5. Bidirectional proxy: `tokio::io::copy_bidirectional`
6. Cleanup on either side disconnect
7. Integration test: full TCP tunnel round-trip

### How to Test
```bash
cargo test -p portalink-server   # Proxy integration tests
```
**Manual validation (the big one!):**
1. Start a local echo server: `nc -l 9000`
2. Start portalink server: `portalink-server --port 4000`
3. Start portalink client: `portalink-client --server 127.0.0.1:4000 --local 127.0.0.1:9000 --type tcp`
4. Connect to server's assigned public port: `nc 127.0.0.1 <assigned_port>`
5. Type text → see it echoed back through the tunnel
6. Verify in server logs: session created, paired, proxying

### Security Checkpoint
- [ ] Session tokens expire after 10s (prevent token hoarding)
- [ ] Expired/invalid tokens rejected with constant-time comparison
- [ ] Max concurrent sessions per tunnel enforced (default: 100)
- [ ] Proxy buffers have bounded size (prevent memory exhaustion)

---

## Phase 6: CLI Interface

**Goal:** User-friendly CLI for both server and client.

### Tasks
1. Server CLI:
   ```
   portalink server --bind 0.0.0.0:4000 --secret <token>
   portalink server --config server.toml
   ```
2. Client CLI:
   ```
   portalink http 8080                          # Quick HTTP tunnel
   portalink tcp 5432 --subdomain mydb          # TCP tunnel
   portalink client --server example.com:4000   # Full config
   ```
3. TOML config file support for both server and client
4. Environment variable overrides (`PORTALINK_SECRET`, etc.)
5. Colored output with connection status
6. Graceful shutdown on SIGINT/SIGTERM

### How to Test
```bash
cargo build --release
./target/release/portalink --help
./target/release/portalink server --help
./target/release/portalink http --help
```
**Manual validation:**
1. `portalink server --secret mysecret` — starts server, shows listening address
2. `portalink http 8080 --server 127.0.0.1:4000 --secret mysecret` — connects, shows public URL
3. Ctrl+C either side → graceful shutdown message

### Security Checkpoint
- [ ] Secret not shown in `--help` or default logs
- [ ] Config file permissions checked (warn if world-readable)
- [ ] Secret can come from env var (avoid shell history exposure)

---

## Phase 7: HTTP Subdomain Routing

**Goal:** Route HTTP traffic by Host header to correct tunnel.

### Tasks
1. Server listens on HTTP port (default 80)
2. Parse `Host` header from incoming HTTP request
3. Extract subdomain: `test.tunnel.example.com` → `test`
4. Look up tunnel by subdomain in registry
5. Forward request to tunnel client, relay response
6. Add `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Forwarded-Host` headers
7. WebSocket upgrade support (passthrough)
8. Return 502 if tunnel offline, 404 if subdomain not found

### How to Test
```bash
cargo test -p portalink-server   # HTTP routing tests
```
**Manual validation:**
1. Start local web server: `python3 -m http.server 8080`
2. Start portalink server on port 4000
3. Start client: `portalink http 8080 --subdomain test`
4. `curl -H "Host: test.localhost" http://127.0.0.1:4000/` → see local server response
5. `curl -H "Host: nonexistent.localhost" http://127.0.0.1:4000/` → 404
6. Kill client → `curl` returns 502

### Security Checkpoint
- [ ] Host header is validated (no null bytes, no path traversal)
- [ ] Request size limits enforced (headers: 8KB, body: configurable)
- [ ] Hop-by-hop headers stripped before forwarding
- [ ] X-Forwarded headers set by server, not trusted from client

---

## Phase 8: TLS with rustls

**Goal:** Encrypt all connections with TLS 1.3.

### Tasks
1. Server-side TLS on control channel (client connects with TLS)
2. Self-signed certificate generation for development (rcgen)
3. SNI-based routing for HTTPS tunnels
4. `TlsTransport` implementation using `tokio-rustls`
5. Certificate hot-reload via `Arc<ArcSwap<ServerConfig>>`
6. Client certificate verification (optional, for mutual TLS)
7. HTTPS listener on port 443 with SNI → tunnel routing

### How to Test
```bash
cargo test -p portalink-core     # TLS transport tests
```
**Manual validation:**
1. Generate dev certs: `portalink server --generate-dev-cert`
2. Start server with TLS: `portalink server --tls-cert cert.pem --tls-key key.pem`
3. Client connects with TLS: `portalink http 8080 --server https://127.0.0.1:4000`
4. Verify with: `openssl s_client -connect 127.0.0.1:4000` — see TLS 1.3 handshake
5. Test HTTPS tunnel: `curl -k https://test.localhost:4443/`

### Security Checkpoint
- [ ] TLS 1.2 minimum enforced (no SSL 3.0, TLS 1.0, 1.1)
- [ ] Strong cipher suites only (AES-256-GCM, ChaCha20-Poly1305)
- [ ] Private keys loaded with restrictive file permissions check
- [ ] Certificate expiry monitoring in logs

---

## Phase 9: QUIC Transport

**Goal:** Add QUIC as primary transport with TCP fallback.

### Tasks
1. `QuicTransport` implementation using quinn
2. QUIC server endpoint on UDP/443
3. Native QUIC stream multiplexing (NO yamux over QUIC)
4. Map each data channel to a QUIC bidirectional stream
5. 0-RTT reconnection (only for non-sensitive operations)
6. Connection migration support
7. Fallback: race QUIC vs TCP, use whichever connects first
8. Benchmark: compare throughput/latency vs TCP transport

### How to Test
```bash
cargo test -p portalink-core     # QUIC transport tests
cargo bench                       # Performance comparison
```
**Manual validation:**
1. Start server with QUIC enabled: `portalink server --quic`
2. Client connects via QUIC: `portalink http 8080 --transport quic`
3. Verify QUIC in use from logs: "QUIC connection established"
4. Block UDP (simulate firewall): client falls back to TCP
5. Switch Wi-Fi network during active tunnel → connection survives (migration)

### Security Checkpoint
- [ ] 0-RTT never used for auth credentials (replay attack prevention)
- [ ] QUIC certificate validation matches TLS rules
- [ ] Connection migration validates source address

---

## Phase 10: ACME Auto-TLS

**Goal:** Automatic wildcard TLS certificates via Let's Encrypt.

### Tasks
1. Integrate `instant-acme` for ACME protocol
2. DNS-01 challenge support (required for wildcards)
3. Cloudflare DNS API integration for challenge response
4. Issue `*.tunnel.example.com` wildcard certificate
5. Certificate storage on disk (encrypted at rest)
6. Auto-renewal when < 30 days to expiry
7. Hot-swap certificates without server restart
8. Let's Encrypt staging environment for testing

### How to Test
```bash
cargo test -p portalink-server   # ACME integration tests (against staging)
```
**Manual validation:**
1. Configure Cloudflare API token
2. Start server: `portalink server --acme --domain tunnel.example.com --cloudflare-token <token>`
3. Server logs: "Requesting wildcard certificate..."
4. Server logs: "Certificate issued, valid until <date>"
5. `curl https://test.tunnel.example.com/` — valid TLS certificate (green lock)
6. Wait for renewal check — logs: "Certificate valid for 89 days, skipping renewal"

### Security Checkpoint
- [ ] ACME account key stored with restrictive permissions
- [ ] Cloudflare API token never logged
- [ ] Certificate private key encrypted at rest
- [ ] Staging environment used for all testing (avoid rate limits)

---

## Phase 11: Production Hardening

**Goal:** Rate limiting, connection limits, metrics, reconnection.

### Tasks
1. Rate limiting on public endpoints (token bucket per IP)
2. Connection limits per client (configurable)
3. Bandwidth throttling per tunnel (optional)
4. Exponential backoff reconnection (client-side)
5. Prometheus metrics endpoint (`/metrics`)
6. Structured JSON logging option
7. Health check endpoint (`/health`)
8. Graceful drain on shutdown (finish active connections)

### How to Test
```bash
cargo test --workspace            # All tests still pass
```
**Manual validation:**
1. Connect 100+ simultaneous clients → verify rate limiting kicks in
2. Kill server → client reconnects with backoff (check logs for intervals)
3. `curl http://server:9090/metrics` → Prometheus format metrics
4. `curl http://server:9090/health` → `{"status":"ok"}`
5. Send SIGTERM → active tunnels finish, new connections rejected

### Security Checkpoint
- [ ] Rate limiter uses sliding window (not fixed)
- [ ] Rate limit applies before auth (prevent auth brute-force)
- [ ] Metrics endpoint not publicly accessible (bind to localhost or auth)
- [ ] Health endpoint doesn't leak internal state

---

## Phase 12: Cross-Platform Build & Distribution

**Goal:** Automated builds for all platforms, install scripts.

### Tasks
1. GitHub Actions CI/CD pipeline
2. Cross-compilation for 6 targets (Linux x64/arm64, macOS x64/arm64, Windows x64, Linux armv7)
3. Binary size optimization (target < 3MB)
4. Docker scratch image
5. Homebrew tap
6. Install script: `curl -fsSL https://get.portalink.dev | sh`
7. SHA256 checksums for all artifacts

### How to Test
```bash
# Local
cargo build --release --target x86_64-unknown-linux-musl
ls -la target/x86_64-unknown-linux-musl/release/portalink   # Check size

# CI
git tag v0.1.0 && git push --tags    # Triggers release workflow
```
**Manual validation:**
1. Download binary for your platform from GitHub Releases
2. Verify checksum: `sha256sum -c portalink-*.sha256`
3. Run `./portalink --version` — shows version
4. Docker: `docker run portalink server` — starts in container

---

## Dependency Graph

```
Phase 0 (Scaffold)
    ↓
Phase 1 (Protocol)
    ↓
Phase 2 (TCP Transport)
    ↓
Phase 3 (Auth) ──────────────────┐
    ↓                             │
Phase 4 (Registration)           │
    ↓                             │
Phase 5 (Data Channel + Proxy)   │
    ↓                             │
Phase 6 (CLI) ←───────────────────┘
    ↓
Phase 7 (HTTP Routing)
    ↓
Phase 8 (TLS) ──→ Phase 9 (QUIC)
    ↓                  ↓
Phase 10 (ACME)        │
    ↓                  │
Phase 11 (Hardening) ←─┘
    ↓
Phase 12 (Distribution)
```

## Testing Strategy

Every phase follows this validation flow:

1. **`cargo fmt --check`** — code is formatted
2. **`cargo clippy --workspace -- -D warnings`** — zero warnings
3. **`cargo test --workspace`** — all tests pass
4. **Manual validation** — follow the steps listed in each phase
5. **Security checklist** — every checkbox ticked before moving on
