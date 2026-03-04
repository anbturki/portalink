# Ngrok Alternative: Required Features

> Evidence-based research compiled from 40+ existing tunnel projects, ngrok's documented architecture, and successful open-source alternatives.

## Sources Analyzed

- [ngrok - How ngrok works](https://ngrok.com/docs/how-ngrok-works)
- [How on Earth Does Ngrok Work - Medium](https://medium.com/@sikehish/how-on-earth-does-ngrok-work-eb00b3af8ebf)
- [awesome-tunneling - GitHub](https://github.com/anderspitman/awesome-tunneling) (40+ projects)
- [rathole - GitHub](https://github.com/rathole-org/rathole)
- [bore - GitHub](https://github.com/ekzhang/bore)
- [frp - GitHub](https://github.com/fatedier/frp)
- [chisel - GitHub](https://github.com/jpillora/chisel)
- [FRP vs Rathole vs ngrok - xTom](https://xtom.com/blog/frp-rathole-ngrok-comparison-best-reverse-tunneling-solution/)
- [TLS Tunnels - Pinggy](https://pinggy.io/blog/tls_tunnels/)
- [Building HTTPS Tunnel in 300 Lines of Go](https://nexo.sh/posts/building-your-own-https-tunnel/)
- [Top 10 Ngrok Alternatives 2026 - Pinggy](https://pinggy.io/blog/best_ngrok_alternatives/)

---

## 1. Must-Have (Core Tunneling)

### 1.1 TCP Tunnel Forwarding

- **What:** Forward arbitrary TCP traffic from a public endpoint to a local service behind NAT.
- **Why:** Foundation of all tunneling. Every single tunnel project in the ecosystem supports this as the base layer.
- **Evidence:** ngrok, frp, rathole, bore, chisel, sish, piko — all implement TCP forwarding as their primary function.

### 1.2 Control Connection + Data Connections

- **What:** Client establishes a persistent control channel for commands (auth, tunnel registration, heartbeats). Separate connections carry the actual proxied data.
- **Why:** Separating control from data allows efficient management without interfering with traffic flow.
- **Evidence:** This is [how ngrok works](https://medium.com/@sikehish/how-on-earth-does-ngrok-work-eb00b3af8ebf):
  - Client opens a long-lived TCP connection to the server (control connection).
  - JSON instruction messages are passed: `Auth`, `AuthResp`, `ReqTunnel`, `NewTunnel`, `Ping`, `Pong`.
  - When external traffic arrives, separate data connections are established.
  - bore uses a similar model: control port (7835) for Hello/Accept handshakes, separate TCP streams for data identified by UUID.

### 1.3 Connection Multiplexing

- **What:** Route many logical streams over a single underlying connection to avoid per-request TCP handshake overhead.
- **Why:** Without multiplexing, each proxied request requires a new TCP connection through the tunnel, adding latency and resource consumption.
- **Evidence:**
  - [go-http-tunnel](https://github.com/mmatczuk/go-http-tunnel) uses HTTP/2 for multiplexing.
  - [yamux](https://github.com/hashicorp/yamux) (Yet another Multiplexer) is used by multiple Go tunnel projects — inspired by SPDY, provides stream-oriented multiplexing over a single connection.
  - The [300-line Go tunnel](https://nexo.sh/posts/building-your-own-https-tunnel/) uses yamux over WebSocket: "multiple HTTP requests transparently inside a single WebSocket tunnel."
  - bore uses UUID-based stream pairing instead of traditional multiplexing.

### 1.4 TLS Encryption on the Tunnel

- **What:** All traffic between the tunnel client and server must be encrypted end-to-end.
- **Why:** Tunnel traffic traverses the public internet. Without encryption, all proxied data (including credentials, API keys, etc.) is exposed.
- **Evidence:**
  - ngrok: "all traffic flows through a persistent TLS-encrypted connection."
  - rathole: Supports TLS with certificate validation AND Noise Protocol (symmetric encryption) — eliminates need for self-signed certificates.
  - chisel: Uses SSH encryption under the hood.
  - frp: Supports TLS transport.

### 1.5 Client Authentication

- **What:** Verify that a client is authorized to create tunnels before allocating resources.
- **Why:** Prevent unauthorized tunnel creation, abuse, and resource exhaustion on the server.
- **Evidence:**
  - ngrok: Auth tokens sent during initial handshake via `Auth` message on the control connection.
  - rathole: **Mandatory** per-service tokens (not optional) — each service gets its own token.
  - bore: HMAC challenge-response authentication — "clients verify possession of the secret on each TCP connection by answering random challenges in the form of HMAC codes."

### 1.6 HTTP/HTTPS Tunnel with Subdomain Routing

- **What:** Route `xyz.yourdomain.com` to the correct tunnel by inspecting the HTTP `Host` header or TLS SNI extension.
- **Why:** HTTP is the most common protocol to tunnel. Subdomain routing allows multiple tunnels on a single server IP/port.
- **Evidence:**
  - frp, chisel, sish, pgrok, Pangolin all implement subdomain-based routing.
  - SNI (Server Name Indication) is the [documented standard approach](https://pinggy.io/blog/tls_tunnels/) for TLS tunnel routing: "an extension in the TLS protocol that includes the hostname requested by the client, sent as part of the initial TLS handshake, allowing the tunnel server to determine the appropriate tunnel."
  - LocalXpose supports "wildcard subdomains for an entire localhost site."

### 1.7 Automatic TLS Certificates

- **What:** Auto-provision Let's Encrypt certificates for public-facing HTTPS endpoints via ACME protocol.
- **Why:** Users expect HTTPS without manual certificate management. Wildcard certs cover dynamic subdomains.
- **Evidence:**
  - chisel: "supports auto certs from LetsEncrypt."
  - Pangolin: "automated SSL certificates."
  - tunwg: "end to end encrypted TLS with LetsEncrypt certificates."
  - reverse-tunnel: "Supports Let's Encrypt."
  - Caddy-based approach: [DNS-01 wildcard challenges](https://nexo.sh/posts/building-your-own-https-tunnel/) for automatic SSL on all subdomains.
  - ExposeME (Rust): "automatic Let's Encrypt SSL with wildcard support."

### 1.8 NAT Traversal

- **What:** Allow services behind NAT/firewalls to be accessible from the public internet.
- **Why:** This is the entire purpose of a tunnel tool.
- **Evidence:** The fundamental mechanism is client-initiated outbound connections. As [ngrok docs explain](https://ngrok.com/docs/how-ngrok-works): "If servers can't connect to you, make your machine connect to them first."

### 1.9 CLI Client

- **What:** Simple command-line interface, e.g., `tunnel http 8080` or `tunnel tcp 5432`.
- **Why:** Developer experience. The CLI is the primary user interface.
- **Evidence:** bore (`bore local 8080`), ngrok (`ngrok http 8080`), frp (config file + `frpc`), chisel (`chisel client`) — all provide single-binary CLI clients.

---

## 2. Should-Have (Production Readiness)

### 2.1 UDP Tunnel Support

- **What:** Forward UDP traffic through the tunnel.
- **Why:** Game servers, DNS, VoIP, IoT protocols, and WireGuard all use UDP. This is a notable gap in ngrok.
- **Evidence:**
  - frp supports UDP tunneling.
  - rathole added UDP support in v0.5+.
  - Pinggy supports UDP tunnels.
  - LocalXpose supports "UDP (L4, layer 4 tunneling)."
  - **ngrok notably does NOT support UDP** — this is a competitive differentiator.

### 2.2 Hot Reload of Configuration

- **What:** Add or remove tunnels dynamically without restarting the server or client.
- **Why:** Zero-downtime operations in production.
- **Evidence:** rathole explicitly lists hot reload as a key feature: "Services can be added or removed dynamically by hot-reloading the configuration file."

### 2.3 Reconnection & Keepalive

- **What:** Automatically reconnect when the tunnel connection drops. Use heartbeats to detect dead connections.
- **Why:** Network disruptions are inevitable. Tunnels should self-heal.
- **Evidence:** ngrok uses Ping/Pong heartbeats on the control connection: "when a tunnel is detected to be dead, the server will clean up all of that tunnel's state and the client will attempt to reconnect and establish a new tunnel."

### 2.4 Custom Domain Support

- **What:** Map `api.yourcompany.com` to a tunnel instead of using a randomly generated subdomain.
- **Why:** Stable URLs for webhooks, development integrations, demos.
- **Evidence:** frp, ngrok (paid), chisel, Pangolin, ExposeME all support custom domains.

### 2.5 Connection Limits & Rate Limiting

- **What:** Limit concurrent connections, bandwidth, and request rates per tunnel/user.
- **Why:** Prevent abuse, ensure fair resource allocation on shared servers.
- **Evidence:** Pangolin (recommended for production by awesome-tunneling maintainer) and piko (designed to "serve production traffic") both implement this.

### 2.6 Metrics / Observability

- **What:** Expose connection counts, bandwidth usage, latency, active tunnels.
- **Why:** Operators need visibility into tunnel server health and usage.
- **Evidence:**
  - frp has a built-in dashboard.
  - Pangolin has a management UI.
  - piko is designed for production with observability in mind.

### 2.7 Multi-Platform Binaries

- **What:** Ship binaries for Linux (amd64, arm64), macOS (amd64, arm64), Windows.
- **Why:** Tunnels need to run everywhere — developer laptops, CI servers, ARM devices (Raspberry Pi).
- **Evidence:**
  - rathole ships ~500KB binaries for multiple platforms.
  - bore provides prebuilt binaries and Docker images.
  - All serious tunnel projects ship cross-compiled binaries.

---

## 3. Nice-to-Have (Differentiators)

### 3.1 QUIC Transport

- **What:** Use QUIC instead of (or alongside) TCP as the tunnel transport layer.
- **Why:** QUIC eliminates TCP head-of-line blocking — if one stream's packet is lost, other streams aren't blocked. Faster connection establishment (0-RTT).
- **Evidence:**
  - frp supports QUIC/KCP multiplexing.
  - [rstun](https://github.com/neevek/rstun) uses QUIC over Quinn in Rust.
  - [Research](https://arxiv.org/html/2504.10054v1) shows QUIC improves tunnel performance by eliminating head-of-line blocking.
  - specter uses "QUIC transport."
  - Quinn (Rust QUIC library): "connections can multiplex an arbitrary number of streams, which are cheap to construct and dispose of."

### 3.2 HTTP Traffic Inspection

- **What:** View request/response headers, bodies, and timing for HTTP tunnels. Optionally replay requests.
- **Why:** Primary debugging tool for webhook development.
- **Evidence:**
  - ngrok's signature feature — traffic inspection and replay.
  - portr implements "request inspection/replay features."

### 3.3 P2P Mode

- **What:** When both client and target are reachable, establish a direct connection bypassing the server.
- **Why:** Reduces server bandwidth costs and latency.
- **Evidence:** frp has P2P mode. specter uses DHT for peer discovery. gt "supports peer-to-peer direct connection."

### 3.4 Web Dashboard

- **What:** Visual UI for managing tunnels, viewing metrics, controlling access.
- **Why:** Easier management for non-technical users or team environments.
- **Evidence:** Pangolin, frp, boringproxy all offer WebUI. portr has a JavaScript/Python admin page.

### 3.5 Proxy Protocol Support

- **What:** Preserve the original client IP address through the tunnel using the PROXY protocol header.
- **Why:** Backend services need real client IPs for logging, access control, and rate limiting.
- **Evidence:** Common requirement in reverse proxy setups; frp and production-grade tunnels support this.

### 3.6 WebSocket Transport

- **What:** Use WebSocket as the tunnel transport (instead of raw TCP).
- **Why:** Works through corporate proxies, firewalls, and CDNs that block non-HTTP traffic.
- **Evidence:**
  - wstunnel: "Focus on proxying from behind networks that block certain protocols."
  - rathole: Supports WebSocket transport with optional TLS.
  - chisel: Uses WebSocket transport.
  - sish: "Supports WebSocket tunneling."
  - ExposeME: "exposes local services through WebSocket connections."

---

## Architecture Overview

Based on the common patterns across all analyzed projects:

```
┌─────────────────┐         ┌──────────────────────┐         ┌────────────┐
│   Public User    │ ──────> │    Tunnel Server      │ ──────> │   Tunnel   │
│  (browser, API   │  HTTPS  │                      │ Control │   Client   │
│   client, etc.)  │         │  - TLS termination   │  Conn   │            │
└─────────────────┘         │  - Subdomain routing  │ (muxed) │  - Runs    │
                            │  - ACME certs         │         │    locally │
                            │  - Auth & rate limit  │         │  - Forwards│
                            │  - Multiplexing       │         │    to local│
                            │  - Metrics            │         │    service │
                            └──────────────────────┘         └────────────┘
                                                                    │
                                                              ┌─────┴──────┐
                                                              │ Local      │
                                                              │ Service    │
                                                              │ (port 8080)│
                                                              └────────────┘
```

### Connection Flow

1. **Client** initiates outbound TLS connection to **Server** (bypasses NAT).
2. **Control channel** established — client authenticates, registers tunnels.
3. **Server** allocates subdomain (e.g., `abc123.tunnel.example.com`).
4. **Public user** sends request to `abc123.tunnel.example.com`.
5. **Server** routes request through multiplexed connection to **Client**.
6. **Client** forwards request to local service on specified port.
7. Response flows back through the same path.
