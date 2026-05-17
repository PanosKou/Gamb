# Gamb Gateway

Gamb is a Rust gateway that fronts upstream inference services (for example Ollama-compatible APIs) and applies authentication, policy checks, and basic observability.

## Current Behavior (as implemented)

- HTTP and HTTPS gateway listeners.
- gRPC listener startup path remains available.
- Transparent upstream forwarding to a configured single upstream.
- Preserves request path and query string (including `/api/*` and `/v1/*`).
- Optional bearer-token auth.
- Optional Cloudflare Access assertion header check (`Cf-Access-Jwt-Assertion`).
- Endpoint allowlist/denylist enforcement.
- Request JSON policy checks for guarded endpoints:
  - `/api/generate`
  - `/api/chat`
  - `/api/embed`
  - `/v1/chat/completions`
- Request body-size and prompt/message-size controls.
- `num_ctx` and `num_predict` ceilings.
- `/metrics` endpoint with request/error/latency/active-stream counters.

## Build / Test

```bash
cargo build --release
cargo test
cargo fmt --check
cargo clippy -- -A dead_code -A warnings
```

## Run

```bash
# Uses config.yaml by default
RUST_LOG=info cargo run --release

# Override config path
GATEWAY_CONFIG=/path/to/config.yaml RUST_LOG=info cargo run --release
```

## Example Config

```yaml
http_port: 8080
https_port: 8443
grpc_port: 50051

# Safe defaults: loopback unless explicitly changed
http_bind_addr: "127.0.0.1"
https_bind_addr: "127.0.0.1"
grpc_bind_addr: "127.0.0.1"

auth:
  oidc_providers: []
  # Current implementation checks equality against this configured value.
  cloudflare_jwt_secret: "example-cf-assertion-token"

tls:
  cert_path: "./cert.pem"
  key_path: "./key.pem"

backends: []

consul_url: "http://localhost:8500"
tls_mode: "file"
tls_domain: "example.com"
tls_email: "admin@example.com"

# Optional bearer token (accepted as either exact match or Bearer-prefixed)
bearer_token: "mysecrettoken"

rate_limit_per_sec: 100
rate_limit_burst: 50

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
```

## Docker

```bash
docker build -t gamb .
docker run --rm -p 8080:8080 -p 8443:8443 \
  -e GATEWAY_CONFIG=/app/config.yaml \
  -v $(pwd)/config.yaml:/app/config.yaml \
  -v $(pwd)/cert.pem:/app/cert.pem \
  -v $(pwd)/key.pem:/app/key.pem \
  gamb
```

## Notes

- The gateway strips `Authorization` and `Cf-Access-Jwt-Assertion` before forwarding to upstream.
- Auth, policy, and metrics are enforced only on requests that pass through the HTTP/HTTPS gateway path.
