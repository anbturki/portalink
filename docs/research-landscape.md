# Tunnel Software Landscape Analysis

> Comprehensive survey of existing ngrok alternatives with language, features, and positioning.

## Sources

- [awesome-tunneling - GitHub](https://github.com/anderspitman/awesome-tunneling)
- [Top 10 Ngrok Alternatives 2026 - Pinggy](https://pinggy.io/blog/best_ngrok_alternatives/)
- [Best ngrok Alternatives 2026 - LocalXpose](https://localxpose.io/blog/best-ngrok-alternatives)
- [FRP vs Rathole vs ngrok - xTom](https://xtom.com/blog/frp-rathole-ngrok-comparison-best-reverse-tunneling-solution/)
- [ngrok Alternatives - Tailscale](https://tailscale.com/learn/ngrok-alternatives)

---

## 1. Market Leaders

### ngrok (Commercial)

- **Language:** Go (v1 open source), closed source (v2+)
- **Protocols:** HTTP, HTTPS, TCP, TLS
- **Notable gaps:** No UDP support
- **Pricing:** Freemium (1 agent, 1 domain free; paid plans for more)
- **Strengths:** Ease of use, global infrastructure, traffic inspection/replay, enterprise features (OAuth, SAML, OIDC, SSO, RBAC)
- **Weaknesses:** Closed source, rate limits on free tier, no UDP, no self-hosting

### Cloudflare Tunnel

- **Type:** Managed service (free)
- **Strengths:** No bandwidth limits on free plan, integrates with Cloudflare DNS, CDN, WAF
- **Weaknesses:** Requires Cloudflare DNS, vendor lock-in, not self-hostable

---

## 2. Open-Source Alternatives by Language

### Go Projects

| Project     | Stars   | License    | Protocols              | Key Feature                          | Status     |
| ----------- | ------- | ---------- | ---------------------- | ------------------------------------ | ---------- |
| frp         | 100k+   | Apache 2.0 | HTTP/S, TCP, UDP, QUIC | Most feature-complete, P2P mode      | Active     |
| chisel      | 13k+    | MIT        | HTTP/S, TCP (SSH-based)| Auto LetsEncrypt, SSH transport      | Active     |
| Pangolin    | 5k+     | N/A        | HTTP/S                 | Full management UI, IAM, SSL        | Active     |
| sish        | 4k+     | MIT        | HTTP/S, TCP, SSH, WS   | SSH-based, WebSocket support         | Active     |
| piko        | 2k+     | MIT        | HTTP/S, TCP            | Designed for production traffic      | Active     |
| gost        | N/A     | MIT        | TCP, UDP, TAP/TUN      | Load balancing, Web API              | Active     |
| zrok        | N/A     | Apache 2.0 | HTTP, Files            | Zero trust (built on OpenZiti)       | Active     |
| specter     | N/A     | MIT        | QUIC, TCP, UDP         | DHT-based, QUIC transport            | Active     |
| mmar        | N/A     | AGPL-3.0   | HTTP/S                 | Zero-dependency                      | Active     |

### Rust Projects

| Project     | Stars   | License    | Protocols              | Key Feature                          | Status     |
| ----------- | ------- | ---------- | ---------------------- | ------------------------------------ | ---------- |
| rathole     | 10k+    | Apache 2.0 | TCP, UDP               | 2x throughput over frp, ~500KB binary | Active     |
| bore        | 8k+     | MIT        | TCP                    | 400 lines, dead simple               | Active     |
| wstunnel    | 7k+     | Apache 2.0 | WebSocket              | Bypass protocol-blocking networks    | Active     |
| tunnelto    | 900+    | MIT        | HTTP/S                 | Subdomain routing                    | Active     |
| ExposeME    | New     | N/A        | HTTP (WebSocket)       | tokio + hyper 1.x + rustls          | New        |
| YTunnel     | N/A     | MIT        | HTTPS (via CF)         | Cloudflare Tunnels with TUI          | Active     |

### Other Languages

| Project       | Language   | Stars   | Protocols    | Note                              |
| ------------- | ---------- | ------- | ------------ | --------------------------------- |
| PageKite      | Python     | N/A     | HTTP/S, TCP  | Comprehensive, but Python         |
| Tunnelmole    | TypeScript | N/A     | HTTP/S       | Optionally self-hostable          |
| pipenet       | TypeScript | MIT     | HTTP/S, WS, TCP | Bundles SDK and CLI            |
| h2tunnel      | Node.js    | MIT     | HTTP/2, TCP  | Zero dependencies, HTTP/2 mux    |
| Port Buddy    | Java       | Apache 2.0 | HTTP, TCP, UDP | GUI available                 |
| expose        | PHP        | N/A     | HTTP/S       | PHP ecosystem                     |
| gsocket       | C          | N/A     | TCP          | P2P, very low-level               |

---

## 3. Architecture Patterns Observed

### Transport Layer Choices

| Transport        | Projects Using It                    | Pros                                 | Cons                          |
| ---------------- | ------------------------------------ | ------------------------------------ | ----------------------------- |
| Raw TCP + TLS    | ngrok, rathole, frp, bore            | Simple, efficient, low overhead      | Blocked by some firewalls     |
| WebSocket        | chisel, wstunnel, sish, ExposeME     | Works through HTTP proxies/firewalls | Slight overhead vs raw TCP    |
| SSH              | chisel, sish, pgrok, SirTunnel       | Built-in encryption, well-understood | Heavier than needed           |
| HTTP/2           | go-http-tunnel                       | Built-in multiplexing                | HTTP framing overhead         |
| QUIC             | frp, specter, rstun                  | No head-of-line blocking, 0-RTT      | Newer, UDP may be blocked     |
| WireGuard        | tunwg, Wiredoor, docker-wireguard    | Strong encryption, kernel performance | VPN not tunnel (different scope) |

### Multiplexing Approaches

| Approach          | Projects          | Description                              |
| ----------------- | ----------------- | ---------------------------------------- |
| yamux             | Go tunnel, koding | Stream-oriented multiplexing over single connection |
| HTTP/2 streams    | go-http-tunnel    | Leverage HTTP/2's built-in multiplexing  |
| QUIC streams      | frp, rstun        | Native multiplexing without head-of-line blocking |
| UUID pairing      | bore              | Each connection identified by UUID, matched at server |
| Custom protocol   | ngrok, rathole    | Purpose-built for tunnel-specific needs  |

### Authentication Methods

| Method                | Projects                    |
| --------------------- | --------------------------- |
| Auth token            | ngrok, frp                  |
| Per-service token      | rathole                     |
| HMAC challenge-response | bore                      |
| SSH keys              | chisel, sish, SirTunnel     |
| OAuth/OIDC            | ngrok (enterprise), Pangolin |

---

## 4. Feature Gap Analysis (What ngrok Lacks)

These are features that existing alternatives provide but ngrok does not, representing potential differentiators:

| Feature              | Who Has It                          | ngrok Status |
| -------------------- | ----------------------------------- | ------------ |
| UDP tunneling        | frp, rathole, Pinggy, LocalXpose    | Not supported |
| Self-hostable        | frp, rathole, bore, chisel, etc.    | Not available (SaaS only) |
| QUIC transport       | frp, specter                        | Not available |
| P2P mode             | frp, specter, gt                    | Not available |
| Open source          | All alternatives listed             | Closed source (v2+) |
| Free unlimited usage | frp, rathole, bore (self-hosted)    | Freemium with limits |
| WebSocket transport  | wstunnel, chisel, sish              | Not configurable |

---

## 5. Production Readiness Indicators

Based on the awesome-tunneling maintainer's recommendations and community feedback:

**Recommended for production:**
- **Pangolin** — full management UI, IAM, automated SSL
- **frp** — battle-tested, 100k+ stars, comprehensive feature set
- **piko** — explicitly designed to serve production traffic

**Recommended for simplicity:**
- **bore** — minimal, 400 lines, "that's all it does"
- **SirTunnel** — 0-config, leverages Caddy + OpenSSH

**Performance-focused:**
- **rathole** — Rust, 2x throughput over frp, minimal resource usage
- **gt** — focus on performance, P2P support

---

## 6. Competitive Positioning for a New Project

Based on this landscape analysis, a new tunnel project should target a gap. The current landscape shows:

| Niche                          | Current Leader | Opportunity                          |
| ------------------------------ | -------------- | ------------------------------------ |
| Feature-complete (Go)          | frp            | Hard to compete on features alone    |
| Minimal simplicity (Rust)      | bore           | bore lacks HTTP routing, TLS certs   |
| Performance (Rust)             | rathole        | rathole lacks HTTP subdomain routing |
| Production management          | Pangolin       | Pangolin is Go-only, heavy           |
| QUIC-native tunnel             | None dominant  | Wide open — no Rust QUIC tunnel with HTTP routing |

**The clearest gap:** A Rust tunnel that combines rathole's performance with bore's simplicity, adds HTTP subdomain routing + auto TLS, and uses QUIC as a modern transport option. No existing project fills this exact niche.
