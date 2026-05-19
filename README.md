# Gamb Gateway

Gamb is a Rust service gateway focused on authenticated HTTP/HTTPS forwarding to a configured upstream, guarded model endpoints, and lightweight observability.

## Key Features

- **HTTP/HTTPS Upstream Forwarding** - Proxies requests to `proxy.upstream`, or `GAMB_UPSTREAM` / `http://127.0.0.1:11434` when unset.
- **Bearer and Cloudflare Access Checks** - Validates configured inbound auth headers before proxying.
- **Header Stripping** - Removes `Authorization` and `Cf-Access-Jwt-Assertion` before forwarding to the upstream service.
- **Guarded Model Endpoints** - Applies endpoint allow/deny lists, body size limits, model allowlists, and prompt/context limits on supported JSON APIs.
- **Streaming Response Forwarding** - Streams upstream response chunks to the client and tracks upstream send/read failures.
- **Basic Metrics** - Exposes request, upstream error, active stream, and latency counters at `/metrics`.
- **TLS Termination** - Serves HTTPS with rustls.
- **gRPC Echo Proxy** - Routes Echo RPCs by `service-name` metadata through the backend registry.

---

## Prerequisites

- **Rust** 1.70+ (install via [rustup](https://rustup.rs/))
- **OpenSSL** development libraries (for some dependencies)

```bash
# Ubuntu/Debian
sudo apt-get install pkg-config libssl-dev

# macOS
brew install openssl
```

---

## CLI Commands

### Build

```bash
# Debug build
cargo build

# Release build (optimized)
cargo build --release
```

### Run

```bash
# Run with debug logging
RUST_LOG=info cargo run

# Run release build
RUST_LOG=info cargo run --release

# Or run the binary directly
RUST_LOG=info ./target/release/gamb
```

### Test

```bash
# Run all tests
cargo test

# Run tests with output
cargo test -- --nocapture
```

### Format

```bash
# Check formatting
cargo fmt --check

# Apply formatting
cargo fmt
```

### Lint

```bash
# Run clippy linter
cargo clippy

# Run clippy with warnings as errors
cargo clippy -- -D warnings
```

### Clean

```bash
# Remove build artifacts
cargo clean
```

---

## Configuration

Create a `config.yaml` file in the project root:

```yaml
# HTTP listening port
http_port: 8080

# Optional ports (defaults shown)
# https_port: 8081      # defaults to http_port + 1
# grpc_port: 50051
# tcp_port: 9100
# udp_port: 9200

# Authentication
auth:
  oidc_providers:
    - name: github
      issuer_url: "https://token.actions.githubusercontent.com"
      audience: "https://github.com/org"
  # cloudflare_jwt_secret: "expected-cf-access-jwt-assertion"

# TLS certificates
tls:
  cert_path: "./cert.pem"
  key_path: "./key.pem"

# Backend services used by the gRPC registry and lower-level proxy modules
backends:
  - name: api
    protocol: http
    address: "http://127.0.0.1:9000"
    routes: ["/api"]
  - name: grpc_backend
    protocol: grpc
    address: "http://127.0.0.1:50052"
    routes: []
  - name: tcpservice
    protocol: tcp
    address: "127.0.0.1:9100"
    routes: []

# Service discovery (planned)
consul_url: "http://localhost:8500"
tls_mode: "file"
tls_domain: "example.com"
tls_email: "admin@example.com"

# Bearer token for authentication
bearer_token: "Bearer mysecrettoken"

# HTTP/HTTPS proxy behavior
proxy:
  upstream: "http://127.0.0.1:11434"
  endpoint_allowlist: []
  endpoint_denylist:
    - "/api/pull"
    - "/api/create"
    - "/api/copy"
    - "/api/push"
    - "DELETE /api/delete"
  model_allowlist: []
  max_body_bytes: 1048576
  max_prompt_chars: 32000
  max_num_ctx: 32768
  max_num_predict: 4096

# Rate limiting
rate_limit_per_sec: 100
rate_limit_burst: 50
```

---

## TLS Certificates

Generate self-signed certificates for development:

```bash
openssl req -x509 -newkey rsa:4096 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=localhost"
```

---

## Quick Start

### 1. Build and run Gamb

```bash
# Generate TLS certificates
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=localhost"

# Build and run
RUST_LOG=info cargo run
```

### 2. Start a test backend

```bash
# Simple Python HTTP server on port 8087
python3 -m http.server 8087
```

### 3. Test the proxy

Set `proxy.upstream: "http://127.0.0.1:8087"` in `config.yaml` when using the Python backend above.

```bash
# Without auth (should return 401)
curl http://localhost:8080/hello
# Returns: 401 Unauthorized

# With auth (proxies to proxy.upstream; auth headers are not forwarded upstream)
curl -H "Authorization: Bearer mysecrettoken" http://localhost:8080/hello
# Returns response from http://127.0.0.1:8087/hello

# Metrics are exposed without proxy auth
curl http://localhost:8080/metrics
```

---

## Docker

### Build

```bash
docker build -t gamb .
```

### Run

```bash
docker run -p 8080:8080 -p 8081:8081 \
  -v $(pwd)/config.yaml:/app/config.yaml \
  -v $(pwd)/cert.pem:/app/cert.pem \
  -v $(pwd)/key.pem:/app/key.pem \
  gamb
```

---

## Kubernetes (Helm)

```bash
helm install gamb ./chart \
  --set image.repository=gamb \
  --set image.tag=latest
```

---

## Architecture

```
                    ┌─────────────────────────────────────┐
                    │            Gamb Gateway             │
                    ├─────────────────────────────────────┤
 HTTP :8080  ──────►│  HTTP Proxy (Hyper + reqwest)       │──────► proxy.upstream
HTTPS :8081  ──────►│  ├─ Bearer / CF Access checks       │
                    │  ├─ Header stripping                │
                    │  ├─ Endpoint and JSON policy checks │
                    │  └─ /metrics text counters          │
 gRPC :50051 ──────►│  gRPC Echo Proxy                    │──────► Backend Registry
                    └─────────────────────────────────────┘
```

---

## HTTP/HTTPS Behavior

The HTTP and HTTPS gateways forward every allowed request to `proxy.upstream` while preserving the incoming path and query string. The `backends` route list is not used by HTTP forwarding.

```bash
# Forward to configured upstream
curl -H "Authorization: Bearer mysecrettoken" http://localhost:8080/api/tags

# Test HTTPS (with self-signed cert)
curl -k -H "Authorization: Bearer mysecrettoken" https://localhost:8081/api/tags
```

Inbound `Authorization` and `Cf-Access-Jwt-Assertion` headers are used only for gateway checks and are stripped before the upstream request is sent.

Guarded JSON endpoints are `/api/generate`, `/api/chat`, `/api/embed`, and `/v1/chat/completions`. On those paths, Gamb enforces configured model, prompt, context, and prediction limits after reading the request body.

`/metrics` returns text counters:

```text
gamb_requests_total <count>
gamb_upstream_errors_total <count>
gamb_active_streams <count>
gamb_latency_ms_sum <milliseconds>
```

## gRPC

```rust
// Example gRPC client
let mut client = EchoClient::connect("http://localhost:50051").await?;
let mut request = Request::new(EchoRequest { message: "hello".into() });
request.metadata_mut().insert("service-name", "grpc_service".parse()?);
let response = client.echo(request).await?;
```

---

## Project Structure

```
gamb/
├── src/
│   ├── main.rs              # Entry point, spawns HTTP, HTTPS, and gRPC gateways
│   ├── config.rs            # YAML configuration parsing
│   ├── backend_registry.rs  # Thread-safe service registry
│   ├── http_proxy.rs        # HTTP/HTTPS proxy implementation
│   ├── grpc_service.rs      # gRPC proxy (Echo service)
│   ├── tcp_udp_proxy.rs     # TCP/UDP proxy module, not started by main.rs
│   ├── tls_config.rs        # TLS/rustls configuration
│   ├── middleware.rs        # Tower middleware utilities
│   └── consul_integration.rs # Consul discovery (planned)
├── proto/
│   └── echo.proto           # gRPC service definition
├── chart/                   # Helm chart for Kubernetes
├── Dockerfile
├── Cargo.toml
├── config.yaml
└── README.md
```

---

## Status

| Feature | Status |
|---------|--------|
| HTTP/HTTPS forwarding to `proxy.upstream` | Working |
| Bearer token auth | Working |
| Cloudflare Access assertion check | Working |
| Header stripping before upstream forwarding | Working |
| Guarded endpoint and JSON policy checks | Working |
| Streaming upstream response forwarding | Working |
| `/metrics` text counters | Working |
| TLS Termination | Working |
| gRPC Echo proxying | Working |
| TCP/UDP proxying | Module present, not started by `main.rs` |
| Rate limiting | Config present, not applied in the HTTP gateway |
| OIDC provider validation | Planned |
| Consul Discovery | Planned |
| Hot Config Reload | Planned |

---

## License

Apache-2.0
