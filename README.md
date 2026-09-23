# SOCKS5 proxy server

A minimal SOCKS5 proxy server built on `mio` with a non-blocking event
loop, RFC 1928 message framing, RFC 1929 username/password
authentication, and server-side DNS resolution over DNS-over-TLS.

> **Security warning.** The transport between client and server is
> encrypted with ChaCha20-Poly1305 (see `lib::crypto::tunnel`). This
> protects the SOCKS5 conversation, RFC 1929 credentials, and any
> plaintext payloads against a passive network observer.
>
> It does **not** protect against the server operator. The server sees
> every plaintext byte inside the tunnel — including destinations and
> any unencrypted application protocol. For end-to-end confidentiality,
> the browser must speak HTTPS (or another encrypted protocol) to the
> target. Use `SOCKS5_TUNNEL_ENCRYPTION=off` only when the transport is
> already protected by another layer (SSH, WireGuard, a private
> network).

## Protocol stack

Each layer is defined by a different specification. "SOCKS5" refers only
to the payload framing, not to the socket or the transport.

| Layer | Specification | Provided by |
|---|---|---|
| Socket API | POSIX sockets (not an RFC) | `mio::net::{TcpListener, TcpStream}` |
| TCP | RFC 9293 | OS kernel |
| **Tunnel** | ChaCha20-Poly1305 AEAD, RFC 8439 | `lib::crypto::tunnel` (client and server) |
| **SOCKS5 messages** | **RFC 1928** | `server` |
| SOCKS5 username/password auth | RFC 1929 | `process_handshake` (`Authenticating`), `Socks5AuthConfig` |
| DNS-over-TLS | RFC 7858 | `lib::dns` |

In short:

* the **socket** is POSIX, not an RFC;
* the **transport** is TCP (RFC 9293);
* the **tunnel** is ChaCha20-Poly1305 (RFC 8439), keyed from the
  HMAC secret already shared with the hub;
* the **protocol carried inside the tunnel** is SOCKS5 (RFC 1928),
  with the optional RFC 1929 auth sub-protocol;
* name resolution is done server-side over DoT (RFC 7858).

## Architecture

```
[browser] --SOCKS5(plaintext)--> [client:1080]
|
| ChaCha20-Poly1305 over TCP
v
[server:1082] --TCP--> [target]
```

The client binds `127.0.0.1:1080` and is the only process the browser
needs to know about. The server binds `0.0.0.0:1082` and is the only
process exposed to the network. All SOCKS5 parsing, DNS resolution, and
target connection happen on the server, inside the encrypted tunnel.

## Configuration

| Env | Required | Default | Meaning |
|---|---|---|---|
| `RUST_LOG` | no | `off` | `tracing` filter, e.g. `info`, `debug`, `server=debug`. |
| `SOCKS5_TUNNEL_ENCRYPTION` | no | `on` | Encrypt the transport between client and server with ChaCha20-Poly1305. Set to `off` only when the transport is already protected. |
| `SOCKS5_USERS` | no | — | RFC 1929 users, e.g. `alice:secret,bob:hunter2`. If unset, the server offers no-auth only. |
| `SOCKS5_AUTH_REQUIRED` | no | `false` | If `true`, refuse to start when `SOCKS5_USERS` is empty. |
| `SOCKS5_SIGNATURE_AUTH` | no | `on` | See `lib::services::setup::PeerSetup`. |
| `SOCKS5_RATE_MAX` | no | `1200` | Per-IP connection attempts per window. `0` disables. |
| `SOCKS5_RATE_WINDOW_SECS` | no | `60` | Rate-limit window. |
| `SOCKS5_QUOTA_MAX` | no | `256` | Concurrent connections per IP. `0` disables. |

Listen address: `0.0.0.0:1082`.

### Tunnel handshake

A client opens a TCP connection to the server and sends a plaintext
`Hello`:

```
MAGIC_HELLO(16) || addr_len(1) || address(N) || timestamp(8) || nonce(16) || HMAC-SHA256(32)
```

The HMAC is computed over all preceding bytes using the account's
`ProxySecret`. The server looks up the account by `address`, checks
revocation, expiry, and timestamp freshness (±60 s), verifies the tag,
and replies with:

```
MAGIC_WELCOME(16) || HMAC-SHA256(32)
```


Both sides then derive two independent keys —

```
k_c2s = HMAC(secret, "socks5-tunnel-v1-c2s" || client_nonce)
k_s2c = HMAC(secret, "socks5-tunnel-v1-s2c" || client_nonce)
```

— and every subsequent byte is a frame:

```
u16 ciphertext_len || 12-byte nonce (counter) || ciphertext || Poly1305 tag
```

The nonce counter is strictly monotonic in each direction; the decrypt
side rejects any frame whose nonce is not exactly the next one. This
eliminates reordering and replay without any additional protocol.

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
- **Timeouts** — 20 s handshake/connect (including the tunnel
  handshake), 300 s idle.
- **Buffers** — 1 MB per direction per connection, plus a 2 MB
  ciphertext staging buffer per direction inside the tunnel.
- **Encryption** — ChaCha20-Poly1305 AEAD, RFC 8439, keyed via
  HMAC-SHA256 from the account secret. No additional key material.
- **DNS** — resolved server-side over DNS-over-TLS (Cloudflare + Quad9),
  per RFC 7858. The DNS query is inside the tunnel.
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

The tunnel authenticates and encrypts every connection to the server.
The remaining hardening steps are:

1. Ensure `SOCKS5_SIGNATURE_AUTH` is enabled (it is, by default) and
   that the local account is registered with the hub.
2. Keep `SOCKS5_TUNNEL_ENCRYPTION=on`. Disabling it turns every
   connection into plaintext SOCKS5 and defeats the handshake.
3. Set a low `SOCKS5_RATE_MAX` (e.g. `5`) and small `SOCKS5_QUOTA_MAX`
   (e.g. `4`) so scanners cannot monopolise resources even though
   they cannot authenticate.
4. Add a firewall allowlist for known client IPs.
5. Do **not** rely on RFC 1929 users alone on a public interface:
   `SOCKS5_USERS` credentials are only safe inside the tunnel. The
   tunnel handshake is the authentication mechanism that matters.

Without a shared secret, an unauthenticated client cannot complete the
tunnel handshake and is dropped at the first byte. The server no longer
serves a plaintext SOCKS5 path to unknown clients by default — the
legacy path exists only for clients that begin with `0x05` and are
explicitly expected.

## Client

See [`CLIENT.md`](CLIENT.md). The client is a separate binary that
binds `127.0.0.1:1080`, completes the tunnel handshake with the server,
and forwards the browser's plaintext SOCKS5 conversation through the
encrypted channel. It performs no SOCKS5 parsing and no DNS
resolution.

## License

See repository.
