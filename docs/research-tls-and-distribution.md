# Research: TLS Certificate Automation & Binary Distribution for a Rust Tunnel Server

> Research date: 2026-03-04

---

## Topic 1: ACME / Let's Encrypt Certificate Automation in Rust

### 1.1 Rust ACME Client Crates Comparison

#### instant-acme

| Field | Value |
|---|---|
| **Crate** | `instant-acme` |
| **Latest Version** | 0.8.4 (main branch); 0.7.2 on lib.rs as of 2025-11-21 |
| **Repository** | [github.com/instant-labs/instant-acme](https://github.com/instant-labs/instant-acme) |
| **Stars** | ~147 |
| **License** | Apache-2.0 |
| **TLS Backend** | rustls (pure Rust, no OpenSSL dependency) |
| **Async Runtime** | Tokio |
| **Challenge Types** | HTTP-01, DNS-01, TLS-ALPN-01 (all three tested against Pebble) |
| **Maintainer Activity** | Active. v0.8 series released mid-2025 with substantial API rework. Dirkjan Ochtman (djc) is the primary maintainer; @cpu joined and contributed integration tests. |
| **Notable Features** | RFC 8555 compliant, ACME Renewal Information (ARI, RFC 9773) support in v0.8, production-proven at Instant Domain Search |

**Verdict**: Best choice for a tunnel server. Pure Rust, supports DNS-01 (required for wildcards), actively maintained, production-proven.

#### rustls-acme

| Field | Value |
|---|---|
| **Crate** | `rustls-acme` |
| **Latest Version** | 0.14.1 (~January 2026) |
| **Repository** | [github.com/FlorianUekermann/rustls-acme](https://github.com/FlorianUekermann/rustls-acme) |
| **Stars** | ~177 |
| **License** | MIT OR Apache-2.0 |
| **TLS Backend** | rustls |
| **Async Runtime** | Runtime-agnostic (no spawned tasks) |
| **Challenge Types** | TLS-ALPN-01 (default), HTTP-01 |
| **Maintainer Activity** | Active. 44 versions published, 407,552 total downloads |
| **Notable Features** | Stream-based API, built-in caching (DirCache), ResolvesServerCertAcme implements rustls::server::ResolvesServerCert |

**Limitation**: Does NOT support DNS-01 challenges. Cannot issue wildcard certificates. Designed for per-domain certs using TLS-ALPN-01.

#### tokio-rustls-acme

| Field | Value |
|---|---|
| **Crate** | `tokio-rustls-acme` |
| **Latest Version** | 0.9.0 |
| **Repository** | [github.com/n0-computer/tokio-rustls-acme](https://github.com/n0-computer/tokio-rustls-acme) |
| **License** | MIT OR Apache-2.0 |
| **TLS Backend** | rustls with ring |
| **Async Runtime** | Tokio only (fork of rustls-acme) |
| **Challenge Types** | TLS-ALPN-01 only |
| **Notable Features** | Focused fork of rustls-acme for Tokio, maintained by n0-computer (iroh project) |

**Limitation**: Same as rustls-acme -- no DNS-01 support, no wildcard certs.

#### acme2

| Field | Value |
|---|---|
| **Crate** | `acme2` |
| **Latest Version** | 0.5.1 |
| **Repository** | [github.com/lucacasonato/acme2](https://github.com/lucacasonato/acme2) |
| **License** | MIT |
| **TLS Backend** | OpenSSL |
| **Async Runtime** | Tokio (via reqwest) |
| **Challenge Types** | DNS-01, HTTP-01 |
| **Maintainer Activity** | **Unmaintained** -- last updated ~4 years ago |
| **Notable Features** | Fully instrumented with tracing, tested against Let's Encrypt and Pebble |

**Verdict**: Supports DNS-01 but abandoned and depends on OpenSSL. Not recommended.

#### acme-lib / acme-micro

| Field | Value |
|---|---|
| **Crate** | `acme-lib` (original) / `acme-micro` (fork by kpcyrd) |
| **Latest Version** | acme-lib 0.9.1 / acme-micro (maintained fork) |
| **Repository** | [github.com/algesten/acme-lib](https://github.com/algesten/acme-lib) / [github.com/kpcyrd/acme-micro](https://github.com/kpcyrd/acme-micro) |
| **License** | MIT |
| **TLS Backend** | OpenSSL |
| **Async Runtime** | Synchronous (blocking) |
| **Challenge Types** | HTTP-01, DNS-01 |
| **Maintainer Activity** | acme-lib last updated ~2 years ago. acme-micro is a maintained fork. |
| **Notable Features** | Simple API, supports multi-domain SANs |

**Verdict**: Synchronous and OpenSSL-dependent. Not suitable for an async tunnel server.

#### acmex (newer entry)

| Field | Value |
|---|---|
| **Crate** | `acmex` |
| **Latest Version** | 0.0.1 |
| **Repository** | [crates.io/crates/acmex](https://crates.io/crates/acmex) |
| **TLS Backend** | rustls |
| **Challenge Types** | TLS-ALPN-01, HTTP-01, DNS-01 |
| **Notable Features** | Works with Let's Encrypt, Google Trust Services, ZeroSSL; file and Redis caching; axum integration |

**Verdict**: Very new (v0.0.1). Promising feature set but unproven. Worth monitoring.

#### Summary Table

| Crate | DNS-01 | HTTP-01 | TLS-ALPN | rustls | OpenSSL | Async | Active |
|---|---|---|---|---|---|---|---|
| **instant-acme** | Yes | Yes | Yes | Yes | No | Yes | Yes |
| **rustls-acme** | No | Yes | Yes | Yes | No | Yes | Yes |
| **tokio-rustls-acme** | No | No | Yes | Yes | No | Yes | Yes |
| **acme2** | Yes | Yes | No | No | Yes | Yes | No |
| **acme-lib** | Yes | Yes | No | No | Yes | No | No |
| **acme-micro** | Yes | Yes | No | No | Yes | No | ~Yes |
| **acmex** | Yes | Yes | Yes | Yes | No | Yes | New |

**Recommendation for tunnel server**: Use `instant-acme` for the ACME protocol layer (DNS-01 wildcard cert acquisition) and potentially `rustls-acme` patterns for the TLS serving layer (SNI resolution, cert hot-swapping).

---

### 1.2 Wildcard Certificate Strategy for Tunnel Subdomains

#### How Existing Tunnel Tools Handle This

**sish** (Go, SSH-based tunnels):
- Uses a wildcard DNS record (`*.tunnel.example.com` -> server IP)
- Delegates TLS to a separate container: `adferrand/docker-letsencrypt-dns` which issues wildcard certs via DNS-01
- Configuration requires DNS provider credentials (Cloudflare, Route53, etc.) via environment variables
- Single wildcard cert covers all tunnel subdomains

**Pangolin** (self-hosted tunneling):
- Uses Traefik as the reverse proxy and TLS termination layer
- Traefik handles automatic Let's Encrypt certificate acquisition
- Uses Wireguard tunnels (via "gerbil" and "newt" components) for the transport layer
- Supports automatic SSL certificates via Let's Encrypt email registration

**chisel** (Go, HTTP tunnel):
- Supports TLS for the tunnel transport itself (server-to-client)
- Does NOT automatically provision certificates for tunnel subdomains
- Users must provide their own TLS certificates or use `--tls-skip-verify`
- No built-in ACME integration

**ngrok** (commercial):
- Issues certs from their own CA for `*.ngrok-free.app` subdomains
- Custom domains require user-provided certificates or ngrok-managed TLS

#### DNS-01 vs HTTP-01 for Wildcard Certificates

**DNS-01 is the ONLY method that can issue wildcard certificates.** This is a hard requirement from Let's Encrypt / ACME specification.

- HTTP-01: Proves control of a single FQDN by serving a file at `http://<domain>/.well-known/acme-challenge/<token>`. Cannot issue wildcards.
- DNS-01: Proves control of the DNS zone by creating a TXT record at `_acme-challenge.<domain>`. Required for `*.tunnel.example.com`.
- DNS-PERSIST-01: New challenge type (expected Q2 2026 production). Set a persistent TXT record once, reuse for all renewals. Eliminates repeated DNS API calls.

For a tunnel server using `*.tunnel.example.com`, DNS-01 is mandatory.

#### DNS Provider API Integration

To automate DNS-01 challenges, you need programmatic access to your DNS provider's API. Common providers:

| Provider | API | Notes |
|---|---|---|
| **Cloudflare** | REST API with zone-scoped tokens | Most common, fast propagation (~seconds) |
| **AWS Route53** | AWS SDK | IAM-scoped permissions possible |
| **Google Cloud DNS** | gcloud API | Service account authentication |
| **DigitalOcean** | REST API | Simple token auth |

**Implementation approach**: Build a `DnsChallengeSolver` trait with provider-specific implementations. For Cloudflare:
1. Create API token with `Zone:DNS:Edit` permission scoped to the tunnel domain
2. On DNS-01 challenge: POST TXT record `_acme-challenge.tunnel.example.com` with challenge value
3. Wait for DNS propagation (poll for TXT record resolution)
4. Notify ACME server to validate
5. Delete TXT record after validation

Rust crates for DNS provider APIs:
- `cloudflare` crate for Cloudflare API
- `aws-sdk-route53` for Route53
- Or use direct HTTP requests via `reqwest`

#### Certificate Renewal Automation Patterns

1. **Timer-based renewal**: Check certificate expiry daily; renew when < 30 days remaining (for 90-day certs) or < 15 days (for 45-day certs)
2. **ACME Renewal Information (ARI)**: RFC 9773 support in instant-acme v0.8. The ACME server tells you the ideal renewal window. Preferred approach.
3. **Cert caching**: Store certs and private keys on disk (encrypted). On restart, load existing valid certs instead of re-issuing.
4. **Hot reload**: When a new cert is acquired, swap it into the running rustls `ServerConfig` without downtime using `Arc<RwLock<ServerConfig>>` or rustls's `ResolvesServerCert` trait.

#### Let's Encrypt Rate Limits

| Limit | Value | Notes |
|---|---|---|
| Certificates per Registered Domain | 50 per week | e.g., 50 certs for `*.example.com` per week |
| Duplicate Certificate | 5 per week | Same exact set of domain names |
| Failed Authorizations | 5 per hour per account | Per identifier, refills 1 per 12 min |
| New Orders | 300 per 3 hours | Per account |
| Renewals | Do NOT count against cert limit | Renewals are exempt |
| Staging Environment | Much higher limits | Use for testing |

**Strategy for a tunnel server**: Use a single wildcard certificate (`*.tunnel.example.com`) which covers ALL tunnel subdomains. This means you only need ONE certificate (and ONE renewal per cycle), staying well within rate limits. The wildcard cert approach is far superior to per-subdomain certs.

**Upcoming changes**:
- May 2026: Optional 45-day certificate lifetime (opt-in `tlsserver` ACME profile)
- February 2027: Default shifts to 64-day certs
- February 2028: Default shifts to 45-day certs
- DNS-PERSIST-01 expected in production Q2 2026 (set DNS record once, renew forever)

---

### 1.3 TLS Termination Architecture

#### Where to Terminate TLS

For a tunnel server, there are two main approaches:

**Option A: Edge Termination (Recommended)**
```
Client Browser                    Tunnel Server                    Tunnel Client
    |                                  |                                |
    |---TLS (*.tunnel.example.com)---->|                                |
    |                           [TLS Termination]                       |
    |                           [SNI Inspection]                        |
    |                           [Route to tunnel]                       |
    |                                  |---plain HTTP over tunnel------>|
    |                                  |                          [local service]
```
- Server holds the wildcard cert private key
- Server inspects SNI to determine which tunnel to route to
- Simpler client implementation (no TLS handling needed)
- Server can inspect/modify HTTP headers (add X-Forwarded-For, etc.)

**Option B: TLS Passthrough**
```
Client Browser                    Tunnel Server                    Tunnel Client
    |                                  |                                |
    |---TLS ClientHello (SNI)--------->|                                |
    |                           [SNI Peek only]                         |
    |                           [Route by SNI]                          |
    |                                  |---raw TLS bytes over tunnel--->|
    |                                  |                          [TLS termination]
    |                                  |                          [local service]
```
- Server only peeks at SNI, never decrypts
- Each tunnel client needs its own cert (or receives the wildcard cert)
- More complex client, but true end-to-end encryption

**Recommendation**: Edge termination (Option A) for an ngrok alternative. This is what ngrok, sish, and Pangolin all do.

#### SNI-Based Routing Implementation with rustls

rustls provides the `ResolvesServerCert` trait for SNI-based certificate selection:

```rust
use rustls::server::{ClientHello, ResolvesServerCert};
use rustls::sign::CertifiedKey;
use std::sync::Arc;

struct TunnelCertResolver {
    // Single wildcard cert for *.tunnel.example.com
    wildcard_cert: Arc<CertifiedKey>,
}

impl ResolvesServerCert for TunnelCertResolver {
    fn resolve(&self, client_hello: ClientHello<'_>) -> Option<Arc<CertifiedKey>> {
        let sni = client_hello.server_name()?;

        // Verify the SNI matches our tunnel domain pattern
        if sni.ends_with(".tunnel.example.com") {
            // Extract tunnel ID from subdomain for routing
            let tunnel_id = sni.strip_suffix(".tunnel.example.com")?;
            // Return the wildcard cert (same cert for all subdomains)
            Some(self.wildcard_cert.clone())
        } else {
            None
        }
    }
}
```

rustls also provides `ResolvesServerCertUsingSni` for mapping specific domain names to certificates, but for a wildcard cert scenario, a custom `ResolvesServerCert` implementation is needed.

**SNI extraction for routing** (without TLS termination):
```rust
// Peek at TLS ClientHello to extract SNI without full handshake
// The SNI appears in the first TLS record (ClientHello)
// Libraries like `tls-parser` or manual parsing of the ClientHello can extract it
```

#### How rustls Handles Certificate Selection Per-Domain

1. `ServerConfig` accepts a `Box<dyn ResolvesServerCert>` via `.with_cert_resolver()`
2. On each incoming TLS connection, rustls calls `resolve()` with the `ClientHello`
3. The `ClientHello` contains the SNI (`server_name()`) and supported cipher suites
4. Your resolver returns the appropriate `CertifiedKey` (cert chain + private key)
5. For a tunnel server with one wildcard cert, the resolver always returns the same cert but uses the SNI for routing decisions

**Certificate hot-swapping pattern**:
```rust
struct HotSwapResolver {
    current_cert: ArcSwap<CertifiedKey>,  // Lock-free atomic swap
}

// When certificate is renewed:
resolver.current_cert.store(Arc::new(new_cert));
// All new connections immediately use the new cert, no restart needed
```

---

## Topic 2: Cross-Compilation and Binary Distribution for Rust

### 2.1 Cross-Compilation Targets for a CLI Tunnel Client

#### Target Matrix

| Target Triple | Platform | Notes |
|---|---|---|
| `x86_64-unknown-linux-musl` | Linux x86_64 | Static binary, runs on any Linux distro |
| `aarch64-unknown-linux-musl` | Linux ARM64 | AWS Graviton, Apple Silicon VMs, modern ARM servers |
| `armv7-unknown-linux-musleabihf` | Linux ARMv7 | Raspberry Pi 2/3/4 (32-bit mode) |
| `x86_64-apple-darwin` | macOS Intel | Older Macs |
| `aarch64-apple-darwin` | macOS Apple Silicon | M1/M2/M3/M4 Macs |
| `x86_64-pc-windows-msvc` | Windows x86_64 | Requires MSVC linker (CI uses windows-latest runner) |

**Note on musl**: The `musl` targets produce fully static binaries with no libc dependency. This is critical for Linux distribution -- a musl binary runs on Ubuntu, Alpine, Debian, RHEL, etc. without any shared library requirements.

#### cross-rs Tool vs Native Toolchains

**`cross` (cross-rs/cross)** -- Recommended approach:
- Drop-in replacement for `cargo` that uses Docker/Podman containers
- Pre-built containers with cross-compilation toolchains for each target
- Zero setup: `cargo install cross && cross build --target aarch64-unknown-linux-musl`
- Handles sysroot, linker, and C library dependencies automatically

**Native toolchains** (alternative):
- Install target toolchain: `rustup target add x86_64-unknown-linux-musl`
- Requires manually installing the C cross-compiler (e.g., `musl-tools` on Debian)
- More complex for ARM targets (need `gcc-aarch64-linux-gnu`, etc.)
- macOS targets can only be compiled on macOS (Apple SDK requirement)

**GitHub Actions recommendation**: Use `cross` for Linux/ARM targets, native compilation on `macos-latest` for macOS targets, and `windows-latest` for Windows.

**Notable GitHub Actions**:
- `houseabsolute/actions-rust-cross` -- composite action wrapping cross
- `dtolnay/rust-toolchain` -- for installing Rust with specific targets
- `softprops/action-gh-release` -- for uploading binaries to GitHub Releases

---

### 2.2 Binary Size Optimization

#### How rathole Achieves ~500KB Binaries

rathole uses a custom `[profile.minimal]` in Cargo.toml plus feature flags:

```toml
[profile.release]
panic = "abort"
lto = true
codegen-units = 1
strip = true

[profile.minimal]
inherits = "release"
opt-level = "z"
lto = true
codegen-units = 1
```

Build command for minimal size:
```bash
cargo build --profile minimal --no-default-features --features client
```

Key techniques rathole uses:
1. **Feature flags**: Separate `client` and `server` features. Build client-only binary without server code.
2. **Optional TLS**: TLS support (`native-tls`, `rustls`, `noise`) are opt-in features, not compiled by default.
3. **`opt-level = "z"`**: Optimize for size over speed.
4. **Post-build stripping and UPX compression**: After cargo build, run `strip` and `upx` on the binary.

#### Cargo Profile Settings Explained

| Setting | Value | Effect |
|---|---|---|
| `opt-level` | `"z"` | Optimize for minimal binary size (smaller than `"s"` usually) |
| `lto` | `true` or `"fat"` | Link-Time Optimization across all crates; eliminates unused code |
| `codegen-units` | `1` | Single codegen unit allows maximum optimization (slower compile) |
| `strip` | `true` or `"symbols"` | Remove symbol table and debug info from binary |
| `panic` | `"abort"` | Remove unwinding machinery (~10% size reduction) |

**Example optimized Cargo.toml**:
```toml
[profile.release]
opt-level = "z"       # Optimize for size
lto = "fat"           # Full link-time optimization
codegen-units = 1     # Maximum optimization
panic = "abort"       # No unwinding
strip = "symbols"     # Strip debug symbols

[profile.release.build-override]
opt-level = "z"
```

**Expected size impact** (approximate, for a tunnel client):
- Default release build: ~5-10 MB
- With `opt-level=z`, `lto=fat`, `codegen-units=1`: ~2-4 MB
- With `strip=true`: ~1.5-3 MB
- With `panic=abort`: ~1.3-2.5 MB
- With UPX compression: ~500KB-1.5 MB

#### UPX Compression with Rust Binaries

UPX (Ultimate Packer for eXecutables) compresses executables by 50-70%. It works as a self-extracting archive -- the binary decompresses itself into memory at startup.

**Compatibility with Rust**:
- Works with dynamically linked Linux binaries (standard glibc targets)
- Works with macOS binaries
- Works with Windows .exe files
- **Known issue**: UPX CANNOT pack musl-linked static binaries (`CantPackException: bad DT_HASH`)
- **Known issue**: `--brute` and `--ultra-brute` flags cause Rust binaries to crash at runtime

**Workaround for musl + UPX incompatibility**:
- Use standard compression (no `--brute`)
- Or skip UPX for musl targets and rely on Cargo profile optimizations only
- Or use glibc targets where UPX works, at the cost of portability

**Recommendation**: For maximum portability, prefer musl static binaries without UPX. The Cargo profile optimizations alone can produce binaries in the 1-2 MB range, which is acceptable for a tunnel client.

---

### 2.3 Distribution Channels

#### GitHub Releases (CI/CD Pipeline)

The primary distribution channel. Typical workflow:

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags: ["v*"]

jobs:
  build:
    strategy:
      matrix:
        include:
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
            use_cross: true
          - target: aarch64-unknown-linux-musl
            os: ubuntu-latest
            use_cross: true
          - target: armv7-unknown-linux-musleabihf
            os: ubuntu-latest
            use_cross: true
          - target: x86_64-apple-darwin
            os: macos-latest
            use_cross: false
          - target: aarch64-apple-darwin
            os: macos-latest
            use_cross: false
          - target: x86_64-pc-windows-msvc
            os: windows-latest
            use_cross: false
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}
      - name: Build
        run: |
          if [ "${{ matrix.use_cross }}" = "true" ]; then
            cargo install cross
            cross build --release --target ${{ matrix.target }}
          else
            cargo build --release --target ${{ matrix.target }}
          fi
      - name: Package
        run: tar czf tunneling-${{ matrix.target }}.tar.gz -C target/${{ matrix.target }}/release tunneling
      - uses: softprops/action-gh-release@v2
        with:
          files: tunneling-*.tar.gz
```

**Caching strategy**: Cache `~/.cargo/registry`, `~/.cargo/git`, and `target/` directories using `actions/cache@v4` to speed up CI.

#### Homebrew Tap

Create a GitHub repo named `homebrew-tap` with a formula:

```ruby
# Formula/tunneling.rb
class Tunneling < Formula
  desc "Fast, secure tunnel to expose local services"
  homepage "https://github.com/yourorg/tunneling"
  version "0.1.0"

  on_macos do
    if Hardware::CPU.arm?
      url "https://github.com/yourorg/tunneling/releases/download/v0.1.0/tunneling-aarch64-apple-darwin.tar.gz"
      sha256 "HASH_HERE"
    else
      url "https://github.com/yourorg/tunneling/releases/download/v0.1.0/tunneling-x86_64-apple-darwin.tar.gz"
      sha256 "HASH_HERE"
    end
  end

  on_linux do
    if Hardware::CPU.arm?
      url "https://github.com/yourorg/tunneling/releases/download/v0.1.0/tunneling-aarch64-unknown-linux-musl.tar.gz"
      sha256 "HASH_HERE"
    else
      url "https://github.com/yourorg/tunneling/releases/download/v0.1.0/tunneling-x86_64-unknown-linux-musl.tar.gz"
      sha256 "HASH_HERE"
    end
  end

  def install
    bin.install "tunneling"
  end
end
```

User installs with: `brew install yourorg/tap/tunneling`

Automation: `cargo-dist` can auto-generate Homebrew formulas on release.

#### cargo install

Users with Rust toolchain installed can build from source:

```bash
cargo install tunneling
```

Requires publishing to crates.io. Ensure the crate name is available.

#### Docker Scratch Images

Multi-stage Dockerfile for minimal container:

```dockerfile
# Build stage
FROM rust:latest AS builder
RUN rustup target add x86_64-unknown-linux-musl
RUN apt-get update && apt-get install -y musl-tools
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

# Runtime stage
FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/tunneling /tunneling
ENTRYPOINT ["/tunneling"]
```

Alternatively, use `clux/muslrust` or `rust-musl-cross` images for the build stage.

Result: Container image of ~2-5 MB containing only the static binary.

For multi-arch images, use `docker buildx` with `--platform linux/amd64,linux/arm64`.

#### AUR Package

Use `cargo-aur` to generate PKGBUILD automatically:

```bash
cargo install cargo-aur
cargo aur
```

This generates a PKGBUILD and a tarball. Manual PKGBUILD example:

```bash
# PKGBUILD
pkgname=tunneling
pkgver=0.1.0
pkgrel=1
pkgdesc="Fast, secure tunnel to expose local services"
arch=('x86_64' 'aarch64')
url="https://github.com/yourorg/tunneling"
license=('MIT')
makedepends=('cargo')

prepare() {
  export RUSTUP_TOOLCHAIN=stable
  cargo fetch --locked --target "$(rustc -vV | sed -n 's/host: //p')"
}

build() {
  export RUSTUP_TOOLCHAIN=stable
  export CARGO_TARGET_DIR=target
  cargo build --frozen --release --all-features
}

package() {
  install -Dm0755 -t "$pkgdir/usr/bin/" "target/release/$pkgname"
}
```

Upload to AUR with `makepkg --printsrcinfo > .SRCINFO` and push to the AUR git repo.

#### One-Line Install Script

For convenience (curl-pipe-bash pattern):

```bash
curl -fsSL https://get.tunneling.dev | sh
```

The script detects OS/arch, downloads the correct binary from GitHub Releases, and installs to `/usr/local/bin` or `~/.local/bin`. Reference: cargo-binstall and similar projects use this pattern.

---

## Recommendations Summary

### ACME / TLS Stack

1. Use **`instant-acme`** for ACME protocol (DNS-01 wildcard cert acquisition)
2. Use **rustls** for TLS termination with a custom `ResolvesServerCert` implementation
3. Issue a **single wildcard certificate** (`*.tunnel.example.com`) via DNS-01
4. Implement **Cloudflare DNS API** integration for DNS-01 challenges (most common provider, fast propagation)
5. Implement **ARI (RFC 9773)** for smart renewal timing
6. Plan for **DNS-PERSIST-01** adoption (Q2 2026) to simplify DNS-01 automation
7. Plan for **45-day certificate lifetimes** starting 2026-2028

### Binary Distribution

1. Target **6 platforms**: linux x64/arm64/armv7, macOS x64/arm64, Windows x64
2. Use **`cross`** for Linux/ARM compilation, native for macOS/Windows
3. Apply **rathole-style Cargo profiles** for size optimization (target ~1-2 MB)
4. Skip UPX for musl targets due to compatibility issues
5. Primary distribution via **GitHub Releases** with CI/CD
6. Secondary: **Homebrew tap**, **cargo install**, **Docker scratch**, **AUR**

---

## References

### ACME Crates

- [instant-acme on crates.io](https://crates.io/crates/instant-acme)
- [instant-acme GitHub (instant-labs/instant-acme)](https://github.com/instant-labs/instant-acme)
- [instant-acme on lib.rs](https://lib.rs/crates/instant-acme)
- [rustls-acme on crates.io](https://crates.io/crates/rustls-acme)
- [rustls-acme GitHub (FlorianUekermann/rustls-acme)](https://github.com/FlorianUekermann/rustls-acme)
- [rustls-acme docs.rs](https://docs.rs/rustls-acme/latest/rustls_acme/)
- [tokio-rustls-acme GitHub (n0-computer/tokio-rustls-acme)](https://github.com/n0-computer/tokio-rustls-acme)
- [tokio-rustls-acme on crates.io](https://crates.io/crates/tokio-rustls-acme)
- [tokio-rustls-acme docs.rs](https://docs.rs/tokio-rustls-acme/latest/tokio_rustls_acme/)
- [acme2 on crates.io](https://crates.io/crates/acme2)
- [acme2 on lib.rs](https://lib.rs/crates/acme2)
- [acme-lib on crates.io](https://crates.io/crates/acme-lib)
- [acme-lib docs.rs](https://docs.rs/acme-lib/latest/acme_lib/index.html)
- [acme-lib GitHub (algesten/acme-lib)](https://github.com/algesten/acme-lib)
- [acme-client on crates.io](https://crates.io/crates/acme-client)
- [acme-client GitHub (onur/acme-client)](https://github.com/onur/acme-client)
- [ACME keyword crates on crates.io](https://crates.io/keywords/acme)
- [letsencrypt keyword crates on crates.io](https://crates.io/keywords/letsencrypt)

### Let's Encrypt and ACME Protocol

- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)
- [Scaling Rate Limits for a Billion Certificates (Jan 2025)](https://letsencrypt.org/2025/01/30/scaling-rate-limits)
- [Let's Encrypt Challenge Types](https://letsencrypt.org/docs/challenge-types/)
- [Decreasing Certificate Lifetimes to 45 Days (Dec 2025)](https://letsencrypt.org/2025/12/02/from-90-to-45)
- [DNS-PERSIST-01: A New Model for DNS-based Challenge Validation (Feb 2026)](https://letsencrypt.org/2026/02/18/dns-persist-01)
- [Let's Encrypt DNS-PERSIST-01 (APNIC Blog)](https://blog.apnic.net/2025/02/06/lets-encrypt-scales-rate-limits-to-prepare-for-a-billion-active-certificates/)
- [DNS-PERSIST-01 certkit overview](https://www.certkit.io/blog/dns-persist-01)
- [Let's Encrypt Client Implementations](https://letsencrypt.org/docs/client-options/)
- [Let's Encrypt Staging Environment](https://letsencrypt.org/docs/staging-environment/)
- [certkit 45-day certificate overview](https://www.certkit.io/blog/45-day-certificates)

### TLS, Wildcard Certificates, and SNI

- [Provisioning TLS Certificates in Rust with ACME (Shuttle, Feb 2025)](https://www.shuttle.dev/blog/2025/02/06/provisioning-tls-certificates-with-acme-in-rust)
- [ACME Challenges Deep Dive: HTTP-01 vs DNS-01 vs TLS-ALPN-01 (NameSilo)](https://www.namesilo.com/blog/en/domain-security/acme-challenges-deep-dive-http-01-vs-dns-01-vs-tls-alpn-01)
- [What is a Wildcard TLS/SSL Certificate? (DigiCert)](https://www.digicert.com/faq/public-trust-and-certificates/what-is-a-wildcard-certificate)
- [Wildcard SSL Certificate Guide (SSL Insights)](https://sslinsights.com/wildcard-ssl-certificate/)
- [Avoid Dangers of Wildcard TLS Certificates (NSA)](https://media.defense.gov/2021/Oct/07/2002869955/-1/-1/0/CSI_AVOID%20DANGERS%20OF%20WILDCARD%20TLS%20CERTIFICATES%20AND%20THE%20ALPACA%20TECHNIQUE_211007.pdf)

### Tunnel Tools

- [chisel GitHub (jpillora/chisel)](https://github.com/jpillora/chisel)
- [chisel TLS passthrough issue #135](https://github.com/jpillora/chisel/issues/135)
- [rathole GitHub (rathole-org/rathole)](https://github.com/rapiz1/rathole)
- [rathole Cargo.toml (build profiles)](https://github.com/yujqiao/rathole/blob/main/Cargo.toml)
- [rathole Build Guide](https://github.com/rapiz1/rathole/blob/main/docs/build-guide.md)
- [rathole Releases](https://github.com/rapiz1/rathole/releases)

### Cross-Compilation

- [cross-rs GitHub (cross-rs/cross)](https://github.com/cross-rs/cross)
- [rust-musl-cross Docker images](https://github.com/rust-cross/rust-musl-cross)
- [muslrust Docker images (clux/muslrust)](https://github.com/clux/muslrust)
- [rust-musl-builder Docker images (emk)](https://github.com/emk/rust-musl-builder)
- [Rustup Cross-Compilation Guide](https://rust-lang.github.io/rustup/cross-compilation.html)
- [Cross-Compiling Rust in GitHub Actions (dzfrias)](https://dzfrias.dev/blog/deploy-rust-cross-platform-github-actions/)
- [Cross-platform Rust CI/CD with GitHub Actions (ahmedjama)](https://ahmedjama.com/blog/2025/12/cross-platform-rust-pipeline-github-actions/)
- [actions-rust-cross GitHub Action](https://github.com/houseabsolute/actions-rust-cross)
- [Cross-compiling Rust projects in GitHub Actions (urth.org)](https://blog.urth.org/2023/03/05/cross-compiling-rust-projects-in-github-actions/)
- [A Rust cross compilation journey (crafteo)](https://blog.crafteo.io/2024/02/29/my-rust-cross-compilation-journey/)
- [Cross-compiling Rust from ARM to x86-64 (Bryan Burgers)](https://burgers.io/cross-compile-rust-from-arm-to-x86-64)
- [rust-cross-toolchain (taiki-e)](https://github.com/taiki-e/rust-cross-toolchain)

### Binary Size Optimization

- [min-sized-rust guide (johnthagen)](https://github.com/johnthagen/min-sized-rust)
- [Rust Binary Size Optimization Examples (Rust-Trends)](https://github.com/Rust-Trends/size-optimization-rust-binaries)
- [Cargo Profiles Reference](https://doc.rust-lang.org/cargo/reference/profiles.html)
- [Rust Performance Book: Build Configuration](https://nnethercote.github.io/perf-book/build-configuration.html)
- [Optimizing Rust Binary Size (AruSahni)](https://arusahni.net/blog/2020/03/optimizing-rust-binary-size.html)
- [Optimize Rust binaries size with cargo and Semver (oknozor)](https://oknozor.github.io/blog/optimize-rust-binary-size/)
- [Optimizations: Speed vs Size (Embedded Rust Book)](https://docs.rust-embedded.org/book/unsorted/speed-vs-size.html)

### Docker and Container Distribution

- [How to Create Minimal Docker Images for Rust Binaries (oneuptime, 2026)](https://oneuptime.com/blog/post/2026-01-07-rust-minimal-docker-images/view)
- [How to Package Rust Applications Into Minimal Docker Containers (alexbrand)](https://alexbrand.dev/post/how-to-package-rust-applications-into-minimal-docker-containers/)
- [Multi-Stage Docker Builds for Static Rust Binaries (deciduously)](https://dev.to/deciduously/use-multi-stage-docker-builds-for-statically-linked-rust-binaries-3jgd)
- [Deploying a Static Rust App in a Barebones Docker Container (apitman)](https://apitman.com/8/)
- [Run Rust in Docker with musl and scratch image (Rust forum)](https://users.rust-lang.org/t/run-rust-in-docker-musl-compiled-in-a-scratch-image/26494)
