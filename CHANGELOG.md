# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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

[Unreleased]: https://github.com/cryptoishere/socks5-tunnel-pub/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.2.0
[0.1.2]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.1.2
[0.1.1]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.1.1
[0.1.0]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.1.0
