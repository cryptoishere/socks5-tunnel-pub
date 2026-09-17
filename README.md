# SOCKS5 proxy server

A minimal SOCKS5 proxy server built on `mio` with a non-blocking event
loop, RFC 1928 message framing, RFC 1929 username/password
authentication, and server-side DNS resolution over DNS-over-TLS.

> **Security warning.** This server does not encrypt its transport.
> When bound to a non-loopback address without SOCKS5 authentication
> enabled, every byte on the wire — including RFC 1929 credentials — is
> visible to anyone on the path, and the port will be discovered and
> abused by internet scanners within minutes. The server refuses to
> start in that configuration by default; see
> [Configuration](#configuration).

## Protocol stack

Each layer is defined by a different specification. "SOCKS5" refers only
to the payload framing, not to the socket or the transport.

| Layer | Specification | Provided by |
|---|---|---|
| Socket API | POSIX sockets (not an RFC) | `mio::net::{TcpListener, TcpStream}` |
| TCP | RFC 9293 | OS kernel |
| **SOCKS5 messages** | **RFC 1928** | `server` |
| SOCKS5 username/password auth | RFC 1929 | `process_handshake` (`Authenticating`), `Socks5AuthConfig` |
| DNS-over-TLS | RFC 7858 | `lib::dns` |

In short:

* the **socket** is POSIX, not an RFC;
* the **transport** is TCP (RFC 9293);
* the **protocol carried on top** is SOCKS5 (RFC 1928), with the optional
  RFC 1929 auth sub-protocol;
* name resolution is done server-side over DoT (RFC 7858).

## Architecture

```
[client] --SOCKS5--> [server:1082] --TCP--> [target]
```

## Build

Prebuilt binaries are attached to each GitHub release. To build from
source:

**Linux (static, glibc-only):**

```bash
cargo build --release --features socks5-server --target x86_64-unknown-linux-gnu --bin server
```

Artifact:

```
target/x86_64-unknown-linux-gnu/release/server
```

**Windows (MSVC):**

Cross-compiling from Linux uses [`cargo-xwin`](https://github.com/rust-cross/cargo-xwin),
which downloads the Windows SDK and MSVC runtime libraries:

```bash
cargo xwin build --release --features socks5-server --target x86_64-pc-windows-msvc --bin server
```

Artifact:

```
target/x86_64-pc-windows-msvc/release/server.exe
```

## Configuration

| Env | Required | Default | Meaning |
|---|---|---|---|
| `RUST_LOG` | no | `off` | `tracing` filter, e.g. `info`, `debug`, `server=debug`. |
| `SOCKS5_USERS` | no | — | RFC 1929 users, e.g. `alice:secret,bob:hunter2`. If unset, the server offers no-auth only. |
| `SOCKS5_AUTH_REQUIRED` | no | `false` | If `true`, refuse to start when `SOCKS5_USERS` is empty. |
| `SOCKS5_SIGNATURE_AUTH` | no | `on` | See `lib::services::setup::PeerSetup`. |
| `SOCKS5_RATE_MAX` | no | `1200` | Per-IP connection attempts per window. `0` disables. |
| `SOCKS5_RATE_WINDOW_SECS` | no | `60` | Rate-limit window. |
| `SOCKS5_QUOTA_MAX` | no | `256` | Concurrent connections per IP. `0` disables. |

Listen address: `0.0.0.0:1082`.

### Safe default

```bash
RUST_LOG=info \
SOCKS5_USERS=alice:secret \
SOCKS5_AUTH_REQUIRED=true \
SOCKS5_RATE_MAX=1200 \
SOCKS5_RATE_WINDOW_SECS=60 \
SOCKS5_QUOTA_MAX=256 \
/opt/socks5-tunnel/server
```

## Operations

See `DEPLOY.md` for the systemd unit, sandboxing profile,
`LimitNOFILE`, and `EnvironmentFile` layout.

Key behaviours:

- **Graceful shutdown** — `SIGINT` / `SIGTERM` stops accepting new
  connections and drains live ones for up to 30 s. A second signal
  forces exit.
- **Timeouts** — 20 s handshake/connect, 300 s idle.
- **Buffers** — 1 MB per direction per connection.
- **DNS** — resolved server-side over DNS-over-TLS (Cloudflare + Quad9),
  per RFC 7858.
- **Admission control** — per-IP token-bucket rate limit and per-IP
  concurrent-connection quota; both disabled by setting the max to `0`.
- **Public IP monitor** — a background thread watches the host's
  outbound public IP and reports changes to a central endpoint. It runs
  independently of the event loop and cannot block it.

### Running on Windows

Environment variables can be set in PowerShell for a one-off test:

```powershell
$env:RUST_LOG = "info"
$env:SOCKS5_USERS = "alice:secret"
$env:SOCKS5_AUTH_REQUIRED = "true"
.\server.exe
```

Or via a wrapper script `run_server.bat` next to the binary, which
scopes the variables with `setlocal`:

```bat
@echo off
setlocal
set RUST_LOG=info
set SOCKS5_USERS=alice:secret
set SOCKS5_AUTH_REQUIRED=true
set SOCKS5_RATE_MAX=1200
set SOCKS5_QUOTA_MAX=256
server.exe %*
endlocal
```

For a Windows service, use [`nssm`](https://nssm.cc/) so the process
receives `CTRL_C_EVENT` on stop and the graceful-shutdown drain runs
as it does under systemd on Linux:

```cmd
nssm install Socks5Tunnel "C:\socks5-tunnel\server.exe"
nssm set Socks5Tunnel AppDirectory C:\socks5-tunnel
nssm set Socks5Tunnel AppEnvironmentExtra ^
    RUST_LOG=info ^
    SOCKS5_USERS=alice:secret ^
    SOCKS5_AUTH_REQUIRED=true
nssm start Socks5Tunnel
```

## Deploying on a public interface

If the server must be reachable from the internet, at minimum:

1. Enable `SOCKS5_USERS` and `SOCKS5_AUTH_REQUIRED=true`.
2. Set a low `SOCKS5_RATE_MAX` (e.g. `5`) and small `SOCKS5_QUOTA_MAX`
   (e.g. `4`) so scanners cannot monopolise resources.
3. Add a firewall allowlist for your client IPs.
4. Better still: tunnel the plaintext SOCKS5 stream over SSH, WireGuard,
   or another encrypted transport, and keep the server bound to
   loopback.

Without authentication, a public server is found within minutes and
abused within hours as an open relay for spam, scraping, and other
traffic that will get your IP blocklisted.

## License

See repository.
