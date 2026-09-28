# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.3.1] - 2026-09-28

### Security

- **Signature authentication is now fail-closed.** When
  `SOCKS5_SIGNATURE_AUTH=true` (the default), the server only accepts
  clients that complete the encrypted tunnel handshake. Every other
  entry path — plaintext SOCKS5 greetings, RFC 1929 exchanges, NO-AUTH
  greetings, malformed first bytes — is rejected before any state
  machine logic runs.

  This closes the open-proxy fallback that was exercised in a live
  attack on 2026-09-28. Three defects combined to make it possible:

  1. `build_signature_config()` returned `None` when the local account
     was missing or unreadable, and `main()` continued to run with
     `auth.required == false`. The listener then offered and accepted
     NO AUTH to any client.
  2. `PendingConnection` in `TunnelHandshake` fell back to the plaintext
     `Greeting` state whenever the first byte was not the tunnel magic.
     Any client could bypass the tunnel entirely.
  3. The `Greeting` method-selection branch accepted the RFC 1929
     signature sub-protocol outside the tunnel whenever
     `signature_config.is_some()`, exposing signed credentials to a
     passive observer.

- **Startup guard.** `SOCKS5_SIGNATURE_AUTH=true` together with
  `SOCKS5_TUNNEL_ENCRYPTION=off` is now a fatal error. Signature auth
  without a tunnel would send credentials in the clear.

- **Replay protection for signed RFC 1929 credentials is bound to the
  peer IP.** A credential block captured from another host no longer
  validates.

### Changed

- `build_signature_config()` now returns
  `Result<Option<Arc<Socks5SignatureConfig>>, _>`. When the flag is
  enabled and the local account cannot be loaded, the process exits
  with a non-zero status instead of silently downgrading to an
  unauthenticated listener.

- `PendingConnection` carries a new `strict_signature_only: bool` field,
  set from `signature_config.is_some()`. In strict mode:

  - `HandshakeState::TunnelHandshake` returns `PendingAction::Close` on
    any first byte that is not `tunnel::MAGIC_HELLO[0]`. The plaintext
    fallback to `Greeting` is removed.
  - `HandshakeState::Greeting` queues `[0x05, 0xff]` and closes if it is
    ever entered before `tunnel_active` is true. This is defence in
    depth — the previous state should be unreachable.
  - The method-selection expression cannot select `0x00` (NO AUTH) or
    `0x02` (USER/PASS) before the tunnel is established.

- `SOCKS5_SIGNATURE_AUTH` now **means what the documentation always
  said it meant**. Previously the flag defaulted to `true` but silently
  disabled itself on any provisioning error, so a running server could
  be unauthenticated while the operator believed signature auth was on.
  The default remains `true`, and the failure mode is now *refuse to
  start*, not *fall back to open proxy*.

- `is_signature_auth_enabled()` no longer has an `.unwrap_or(true)`
  side-effect that hid the missing-flag case in logs. The env var is
  parsed the same way, but absence is now explicit and logged.

### Added

- Regression tests asserting:

  - a strict-mode listener drops `[0x05, 0x01, 0x00]` (NO-AUTH
    greeting) with zero response bytes;
  - a strict-mode listener drops `[0x05, 0x01, 0x02]` (USER/PASS
    greeting) with zero response bytes;
  - `main()` exits non-zero when `SOCKS5_SIGNATURE_AUTH=true` and no
    account is persisted;
  - `main()` exits non-zero when `SOCKS5_SIGNATURE_AUTH=true` and
    `SOCKS5_TUNNEL_ENCRYPTION=off`.

### Fixed

- The tunnel handshake is now the *only* authentication path when
  signature auth is enabled. Operators who relied on the (broken)
  plaintext fallback for legacy clients must either provision those
  clients for the tunnel or set `SOCKS5_SIGNATURE_AUTH=false` and
  accept the risks documented in `README.md`.

## [0.3.0] - 2026-09-23

### Added

- **Encrypted tunnel between client and server** (`lib::crypto::tunnel`).
  - ChaCha20-Poly1305 AEAD (RFC 8439) framing over TCP.
  - Per-direction keys derived via HMAC-SHA256 from the existing shared
    `ProxySecret` and a fresh 128-bit client nonce, so no new key
    material needs to be provisioned.
  - Plaintext handshake: client sends `Hello` (address, timestamp,
    nonce, HMAC tag), server replies with `Welcome` (HMAC tag over the
    client tag). All bytes after the Welcome are framed ciphertext.
  - Strict monotonic nonce counter per direction. Reordering, replay,
    and truncation are rejected by the AEAD tag check or the counter
    check.
  - Backpressure-aware: `read_remote` stops pulling ciphertext once the
    browser-side output buffer is full, so TCP backpressure propagates
    end-to-end.
  - Half-close is preserved across the tunnel: the client only issues
    `Shutdown::Write` on the remote after the ciphertext pipeline has
    fully drained.

- `SOCKS5_TUNNEL_ENCRYPTION` environment variable (default `on`).
  - When `on`, the server runs the tunnel handshake before the SOCKS5
    state machine and rejects any non-tunnel first byte.
  - When `off`, the server behaves exactly as in `0.2.0`: plaintext
    SOCKS5, HMAC-signed RFC 1929 credentials.

- `chacha20poly1305` dependency (RustCrypto, `alloc` feature only).

### Changed

- **`browser_client` is now an encrypted pipe, not a SOCKS5 chaining
  relay.**
  - The client no longer parses SOCKS5, no longer performs the RFC 1929
    exchange, and no longer forwards a rewritten greeting.
  - It accepts the browser's plaintext SOCKS5 on `127.0.0.1:1080`,
    completes the tunnel handshake with the remote, and then shuttles
    raw bytes through AEAD frames. The remote proxy performs all SOCKS5
    parsing, DNS resolution, and target connection — inside the tunnel.
  - `BrowserClientConfig` now carries `address: String` and
    `secret: ProxySecret` instead of a `CredentialsProvider`.
  - The SOCKS5 greeting the browser sees is the server's — the client
    has no protocol opinions of its own.

- Server connection state machine extended with two handshake states
  (`TunnelHandshake`, `TunnelWelcome`) before `Greeting`. Legacy
  plaintext clients are still served when the first byte is not the
  tunnel magic; a client that starts with `0x05` bypasses the tunnel
  and is handled by the existing SOCKS5 path.

- On the tunnel path, the greeting step forces method `0x00`
  (NO AUTH) once the tunnel handshake succeeds. The tunnel already
  proved identity; the browser's SOCKS5 greeting cannot downgrade it.

### Security

- Network traffic between client and server is now opaque. A passive
  observer sees the client's registered proxy address and a timestamp
  in the plaintext Hello, then ciphertext only. The SOCKS5 greeting,
  CONNECT request, destination hostname (when ATYP=domain), and all
  payloads are inside encrypted frames.

- Plaintext payloads inside the tunnel are still readable by the
  server. The tunnel does not change the trust model for HTTP: only the
  browser's own TLS to the target hides HTTPS payloads from the
  server. The tunnel hides the *transport* from the network, not the
  payload from the proxy operator.

## [0.2.0] - 2026-09-17

### Changed

- Increased the public-IP check interval in `lib::host_ip` from
  **30 seconds** to **5 minutes** (`CHECK_INTERVAL`).
  - The IP monitor previously woke up once every 30 seconds and
    performed a full check cycle: fetch the public IP from the
    external checker pool, and — when the value changed — report the
    new IP to the central endpoint and refresh the persisted proxy
    secret.
  - This produced 120 check cycles per hour per running proxy, each
    one issuing an HTTPS request to `api.ipify.org` (or one of the
    fallback checkers). Across a fleet of proxies that is significant
    outbound traffic and load on the checker endpoints, for a value
    that changes at most a few times a day on a typical connection.
  - The interval is now 300 seconds (5 minutes), reducing the traffic
    by 10× while still detecting an IP change well within the
    freshness window the central endpoint cares about.
  - No other behaviour changes. The first check still runs
    immediately on startup (`next_check = Instant::now()`), so a
    proxy that has just started still reports its IP without waiting
    for the first interval to elapse.

- `ProxySecret` now stores a fixed-size `[u8; 32]` instead of a
  `Vec<u8>`.
  - The length invariant is enforced at the type level: a secret that
    is not exactly 32 raw bytes cannot be constructed.
  - `from_base64` is now the only path that decodes an encoded secret,
    and it rejects any input that does not decode to exactly 32 bytes.
  - `from_raw_bytes` takes `[u8; 32]` directly, so a base64 string
    passed as a byte vector can no longer be mistaken for a raw secret.
  - `as_bytes` returns `&[u8; 32]` rather than `&[u8]`.
  - This fixes the class of HMAC failures where the proxy was signing
    with a base64 string (43 bytes) while the hub was signing with the
    raw secret (32 bytes), producing `invalid signature` on every
    request.

- `build_signature_config` no longer captures a snapshot of the local
  account.
  - Previously it read the account from sled once at startup, built a
    `ProxyAccount` from that value, and stored it inside
    `LocalAccountSource`. Every subsequent verification used that
    startup snapshot, so any later change to the persisted secret — for
    example from the IP monitor's `update` path — was invisible to the
    verifier.
  - It now only performs a fail-fast check that an account exists, and
    constructs a stateless `LocalAccountSource`.
  - The account is no longer cloned into `Socks5SignatureConfig`, and
    the secret is no longer logged.

- `LocalAccountSource` now performs a read-through lookup on every
  verification instead of holding a cached `ProxyAccount`.
  - Each call to `lookup` reads the current account from
    `PeerManagement`, decodes the secret with
    `ProxySecret::from_base64`, and returns a fresh `ProxyAccount`.
  - This makes secret rotations visible to the verifier immediately.
    When the IP monitor updates the account, the next signed request
    is verified against the new secret without a restart.
  - The cost is one sled read per authenticated request, which is
    negligible for a SOCKS5 proxy.

### Fixed

- Signature verification no longer fails with `invalid signature`
  after the account's HMAC secret is updated by the IP monitor or by
  any other code path that writes to the persisted account.
  - Root cause: the verifier held a snapshot of the secret from
    startup and never observed subsequent updates.
  - With the read-through `LocalAccountSource`, verification always
    uses the current persisted value.

- Signature verification no longer fails when the persisted secret was
  written by an older build that used `Vec<u8>`.
  - Root cause: postcard is not self-describing, so a value written by
    the previous struct layout was silently misinterpreted under the
    new layout, producing a 32-byte "secret" that was not the original.
  - With `ProxySecret` fixed to `[u8; 32]`, and the read path going
    through `from_base64`, old records either decode correctly or fail
    explicitly.

## [0.1.2] - 2026-09-16

### Fixed

- Fixed an issue where the return value from a spawned task was stored
  in a block scope. When the block ended, the return value was
  automatically dropped. Dropping the value caused the `IpMonitor` to
  stop, which in turn caused the program to terminate unexpectedly.

## [0.1.1] - 2026-09-16

### Changed

- Replaced `sled::open(db_path())` with a configured `sled::Config`.
  - Configured sled to use `Mode::HighThroughput`.
  - Set the sled cache capacity to 64 MiB.
- Updated rustls from `"0.23.44"` to `"0.23.45"`.
- Moved underlying database initialization behind the `socks5-server`
  feature flag.
- The startup guard is activated when signature authentication is
  enabled via the `SOCKS5_SIGNATURE_AUTH` environment variable.

## [0.1.0] - 2026-09-15

### Added

- `server`: SOCKS5 proxy built on `mio` with a non-blocking event loop.
  - RFC 1928 message framing (greeting, CONNECT, reply).
  - RFC 1929 username/password authentication (`SOCKS5_USERS`).
  - Server-side DNS-over-TLS resolution (RFC 7858, Cloudflare + Quad9).
  - IPv4, IPv6, and domain-name destinations.
  - Graceful shutdown on `SIGINT` / `SIGTERM` with a 30 s drain window.
  - Per-IP rate limit (`SOCKS5_RATE_MAX`, `SOCKS5_RATE_WINDOW_SECS`).
  - Per-IP concurrent-connection quota (`SOCKS5_QUOTA_MAX`).
- `lib::auth`, `lib::dns`, `lib::rate_limit`, `lib::support` modules.
- `lib::host_ip`: public-IP monitor running on its own thread.
- `lib::services::setup::PeerSetup`: startup guard.
- `lib::tracing`: process-wide `tracing-subscriber` initialisation;
  log level controlled by `RUST_LOG`.
- Environment variables `SOCKS5_SIGNATURE_AUTH` and
  `SOCKS5_AUTH_REQUIRED`.
- `README.md`, `CLIENT.md`, `DEPLOY.md`.

### Distribution

- Prebuilt static binaries attached to the `v0.1.0` GitHub release:
  `x86_64-linux-gnu` and `x86_64-pc-windows-msvc`.

[Unreleased]: https://github.com/cryptoishere/socks5-tunnel-pub/compare/v0.3.1...HEAD
[0.3.1]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.3.1
[0.3.0]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.3.0
[0.2.0]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.2.0
[0.1.2]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.1.2
[0.1.1]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.1.1
[0.1.0]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.1.0
