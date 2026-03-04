# Portalink

A self-hosted, open-source tunnel server and client — an alternative to ngrok focused purely on tunneling.

Expose local services to the public internet through secure, encrypted tunnels. Supports HTTP/HTTPS with subdomain routing, raw TCP, and UDP forwarding.

## Status

**Pre-development** — Currently in the research and design phase.

## Goals

- Self-hostable with zero vendor lock-in
- High performance with minimal resource usage
- Simple CLI interface (`portalink http 8080`)
- Automatic TLS certificates via Let's Encrypt
- HTTP subdomain routing and custom domain support
- TCP and UDP tunnel support
- QUIC transport with TCP/WebSocket fallback
- Single static binary for client and server

## Tech Stack

- **Language:** Rust
- **Async runtime:** tokio
- **TLS:** rustls (pure Rust, no OpenSSL dependency)
- **QUIC:** quinn
- **HTTP:** hyper + axum
- **Multiplexing:** QUIC native streams (yamux for TCP fallback)
- **Serialization:** postcard (serde-compatible binary format)
- **CLI:** clap

## Research

The `docs/` directory contains comprehensive, evidence-based research covering:

| Document | Description |
|----------|-------------|
| [Features](docs/research-features.md) | Required features analysis across 3 priority tiers |
| [Landscape](docs/research-landscape.md) | Survey of 40+ existing tunnel projects and gap analysis |
| [Language](docs/research-language.md) | Rust vs Go comparison with benchmarks |
| [Rust Crates](docs/research-rust-crates.md) | 18 crates evaluated with verified versions and compatibility |
| [Protocol Design](docs/research-protocol-design.md) | Source code analysis of bore, rathole, frp, and chisel |
| [QUIC Transport](docs/research-quic-transport.md) | Quinn library, QUIC vs TCP, fallback strategies |
| [TLS & Distribution](docs/research-tls-and-distribution.md) | ACME automation, cross-compilation, binary optimization |

## License

MIT
