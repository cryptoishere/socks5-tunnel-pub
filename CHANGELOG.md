# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.2] - 2026-09-16

### Fixed
Fixed an issue where the return value from a spawned task was stored in a block scope.

When the block ended, the return value was automatically dropped.

Dropping the value caused the IpMonitor to stop, which in turn caused the program to terminate unexpectedly.

## [0.1.1] - 2026-09-16

### Changed

- Replaced `sled::open(db_path())` with a configured `sled::Config`.
  - Configured sled to use `Mode::HighThroughput`.
  - Set the sled cache capacity to 64 MiB.
- Updated rustls from `"0.23.44"` to `"0.23.45"`.
- Moved underlying database initialization behind the `socks5-server` feature flag.
- The startup guard is activated when signature authentication is enabled via the `SOCKS5_SIGNATURE_AUTH` environment variable.

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

[Unreleased]: https://github.com/cryptoishere/socks5-tunnel-pub/compare/v0.1.2...HEAD
[0.1.2]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.1.2
[0.1.1]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.1.1
[0.1.0]: https://github.com/cryptoishere/socks5-tunnel-pub/releases/tag/v0.1.0
