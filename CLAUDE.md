# Portalink - Project Instructions

## Overview

Portalink is a self-hosted tunnel server/client (ngrok alternative) written in Rust.
Security-critical system — all code must follow security best practices.

## Architecture

- **Cargo workspace** with separate crates: `portalink-core`, `portalink-server`, `portalink-client`, `portalink-cli`
- **Transport trait** abstracts TCP, TLS, QUIC, WebSocket — implementations are pluggable
- **Control channel** (persistent) + **Data channels** (per-visitor session)
- **Protocol**: Length-delimited binary frames using `postcard` serialization
- **Wire format**: `[1 byte type tag][4 bytes length][N bytes payload]`

## Tech Stack (Pinned Versions)

| Crate | Version | Purpose |
|-------|---------|---------|
| tokio | 1.49 | Async runtime |
| hyper | 1.7 | HTTP handling |
| axum | 0.8 | HTTP server/routing |
| rustls | 0.23 | TLS |
| tokio-rustls | 0.26 | Async TLS |
| quinn | 0.11 | QUIC transport |
| yamux | 0.13 | Stream multiplexing (TCP fallback) |
| clap | 4.5 | CLI |
| serde | 1.0 | Serialization |
| postcard | 1.1 | Binary wire format |
| toml | 0.9 | Config files |
| tracing | 0.1 | Structured logging |
| tracing-subscriber | 0.3 | Log output |
| instant-acme | 0.8 | ACME certificates |
| rcgen | 0.14 | Certificate generation |

## Rust Conventions

- **Edition**: 2024
- **MSRV**: 1.88 (set by rcgen)
- Use `thiserror` for library errors, `anyhow` only in CLI/binary entry points
- All public types must derive `Debug`
- Use `#[must_use]` on functions that return important values
- Prefer `Arc<T>` over `Rc<T>` — everything must be `Send + Sync`
- Use `tracing` macros (`info!`, `warn!`, `error!`, `debug!`, `trace!`) — never `println!`
- Async code uses `tokio` — never `std::thread` unless justified
- All network I/O must have timeouts — no unbounded waits
- Use `SecretString` or zeroize for sensitive data in memory

## Security Rules

- **NEVER** log secrets, tokens, or private keys — even at `trace!` level
- All authentication uses HMAC-SHA256 with server-generated nonces
- Session tokens are UUID v4, stored in `DashMap` with TTL (10s default)
- TLS 1.2+ minimum — no plaintext control channels in production
- Rate limit all public-facing endpoints
- Validate all incoming message sizes (max frame size: 64KB for control, configurable for data)
- Sanitize subdomain names (alphanumeric + hyphens only, max 63 chars)
- Never trust client-supplied port numbers or addresses without validation
- Use `constant_time_eq` for token/hash comparison — prevent timing attacks

## Code Patterns

### Error Handling
```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ProtocolError {
    #[error("authentication failed")]
    AuthFailed,
    #[error("invalid frame: {0}")]
    InvalidFrame(String),
    #[error("io error: {0}")]
    Io(#[from] std::io::Error),
}
```

### Transport Trait
```rust
#[async_trait]
pub trait Transport: Send + Sync + 'static {
    type Stream: AsyncRead + AsyncWrite + Send + Unpin + 'static;
    type Listener: Send + 'static;

    async fn connect(&self, addr: &str) -> Result<Self::Stream>;
    async fn bind(&self, addr: &str) -> Result<Self::Listener>;
    async fn accept(listener: &Self::Listener) -> Result<Self::Stream>;
}
```

### Message Types
```rust
#[derive(Debug, Serialize, Deserialize)]
pub enum ClientMessage {
    Auth { version: String, token_hash: Vec<u8> },
    RegisterTunnel { tunnel_id: String, tunnel_type: TunnelType, config: TunnelConfig },
    AcceptDataChannel { session_token: String },
    Pong,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum ServerMessage {
    AuthChallenge { nonce: String, server_version: String },
    AuthResult { success: bool, client_id: Option<String>, error: Option<String> },
    TunnelRegistered { tunnel_id: String, public_addr: String },
    OpenDataChannel { session_token: String, tunnel_id: String },
    Ping,
    Error { code: u16, message: String },
}
```

## Testing Conventions

- Unit tests in the same file (`#[cfg(test)] mod tests {}`)
- Integration tests in `tests/` directory of each crate
- Use `tokio::test` for async tests
- Test helpers go in `src/test_utils.rs` (gated behind `#[cfg(test)]`)
- Every phase must include validation steps the developer can run manually
- Use `cargo test`, `cargo clippy`, `cargo fmt --check` before considering anything done

## Implementation Phases

See `docs/implementation-plan.md` for the full breakdown. Each phase:
1. Has a clear scope and deliverable
2. Is independently testable
3. Includes manual validation steps
4. Builds on previous phases without breaking them
