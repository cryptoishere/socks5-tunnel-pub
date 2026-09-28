# SOCKS5 proxy server

A minimal SOCKS5 proxy server built on `mio` with a non-blocking event
loop, RFC 1928 message framing, RFC 1929 username/password
authentication, and server-side DNS resolution over DNS-over-TLS.

> **Security warning.** The transport between client and server is
> encrypted with ChaCha20-Poly1305 (see `lib::crypto::tunnel`). This
> protects the SOCKS5 conversation, RFC 1929 credentials, and any
> plaintext payloads against a passive network observer.
>
> As of `0.3.1`, when `SOCKS5_SIGNATURE_AUTH=true` (the default) the
> **encrypted tunnel is the only accepted entry point**. The server
> refuses to start if it cannot honour that guarantee, and rejects any
> connection whose first byte is not the tunnel magic. See
> [Strict mode](#strict-mode) below.
>
> The tunnel does **not** protect against the server operator. The
> server sees every plaintext byte inside the tunnel — including
> destinations and any unencrypted application protocol. For end-to-end
> confidentiality, the browser must speak HTTPS (or another encrypted
> protocol) to the target. Use `SOCKS5_TUNNEL_ENCRYPTION=off` only when
> the transport is already protected by another layer (SSH, WireGuard,
> a private network) **and** you have set
> `SOCKS5_SIGNATURE_AUTH=false` — the server will otherwise refuse to
> start.

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
| `SOCKS5_TUNNEL_ENCRYPTION` | no | `on` | Encrypt the transport between client and server with ChaCha20-Poly1305. Setting this to `off` while `SOCKS5_SIGNATURE_AUTH=on` is a fatal startup error. |
| `SOCKS5_USERS` | no | — | RFC 1929 users, e.g. `alice:secret,bob:hunter2`. Only used on the non-strict path. Ignored when the tunnel is active. |
| `SOCKS5_AUTH_REQUIRED` | no | `false` | If `true`, refuse to start when `SOCKS5_USERS` is empty. Only meaningful when `SOCKS5_SIGNATURE_AUTH=off`. |
| `SOCKS5_SIGNATURE_AUTH` | no | `on` | **Strict mode.** When `on`, the encrypted tunnel handshake is the only accepted entry point and the server refuses to start if no account is provisioned. See [Strict mode](#strict-mode). |
| `SOCKS5_RATE_MAX` | no | `1200` | Per-IP connection attempts per window. `0` disables. |
| `SOCKS5_RATE_WINDOW_SECS` | no | `60` | Rate-limit window. |
| `SOCKS5_QUOTA_MAX` | no | `256` | Concurrent connections per IP. `0` disables. |

Listen address: `0.0.0.0:1082`.

## Strict mode

When `SOCKS5_SIGNATURE_AUTH=true` — **the default** — the server
enforces the following invariants at startup and per connection.

### At startup (fail-closed)

The process exits with a non-zero status if **any** of these holds:

- the local account cannot be loaded from the peer store (missing,
  unreadable, or corrupt);
- `SOCKS5_TUNNEL_ENCRYPTION` is `off`.

There is no runtime path from "strict mode requested" to "listener up
but unauthenticated". Either the server can honour the policy, or it
does not start.

### Per connection

Every accepted TCP connection must begin with the tunnel `Hello`
(`tunnel::MAGIC_HELLO[0]`). If the first byte is anything else — a
plaintext SOCKS5 version byte `0x05`, a TLS `ClientHello`, a port
scanner probe, or garbage — the socket is closed immediately with **no
response bytes**. The plaintext SOCKS5 state machine is not entered.

Once the tunnel handshake succeeds and the cipher is installed:

- the SOCKS5 greeting must contain `0x00` (NO AUTH) as the only
  accepted method, because identity has already been proven by the
  tunnel HMAC;
- the method-selection branch cannot be coerced into `0x02`
  (USER/PASS) or into a fallback `0x00` before the tunnel is active.

### What is explicitly rejected

| Client behaviour | Result |
|---|---|
| Plaintext greeting `05 01 00` (NO AUTH) | socket closed, no reply |
| Plaintext greeting `05 01 02` (USER/PASS) | socket closed, no reply |
| Plaintext greeting with any other method list | socket closed, no reply |
| Tunnel Hello with stale timestamp (`>60 s` skew) | socket closed |
| Tunnel Hello with bad HMAC | socket closed, logged |
| Tunnel Hello from revoked account | socket closed |
| Tunnel Hello from expired account | socket closed |
| Tunnel Hello from unknown address | socket closed, logged |
| First byte is `0x05` (tunnel magic mismatch) | socket closed, logged |

### Turning strict mode off

`SOCKS5_SIGNATURE_AUTH=false` restores the legacy plaintext path. This
is intended for:

- offline development on a loopback-only listener,
- testing in an isolated lab network,
- deployments where the transport is already protected by SSH,
  WireGuard, or a private network.

It is **not** a supported configuration on a public interface. If you
disable strict mode on a host reachable from the internet, you have an
open proxy (with or without RFC 1929 users — the RFC 1929 password is
sent in the clear on this path).

## Tunnel handshake

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

## Safe default

```bash
RUST_LOG=info \
SOCKS5_SIGNATURE_AUTH=true \
SOCKS5_TUNNEL_ENCRYPTION=on \
SOCKS5_RATE_MAX=1200 \
SOCKS5_RATE_WINDOW_SECS=60 \
SOCKS5_QUOTA_MAX=256 \
/opt/socks5-tunnel/server
```

Note that this configuration does **not** set `SOCKS5_USERS`. Under
strict mode the RFC 1929 path is never reached, and the server
authenticates clients via the tunnel HMAC against the account
provisioned by the hub.

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
$env:SOCKS5_SIGNATURE_AUTH = "true"
$env:SOCKS5_TUNNEL_ENCRYPTION = "on"
.\server.exe
```

Or via a wrapper script `run_server.bat` next to the binary, which
scopes the variables with `setlocal`:

```bat
@echo off
setlocal
set RUST_LOG=info
set SOCKS5_SIGNATURE_AUTH=true
set SOCKS5_TUNNEL_ENCRYPTION=on
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
    SOCKS5_SIGNATURE_AUTH=true ^
    SOCKS5_TUNNEL_ENCRYPTION=on
nssm start Socks5Tunnel
```

## Deploying on a public interface

The tunnel authenticates and encrypts every accepted connection. The
remaining hardening steps are, in priority order:

1. **Keep strict mode on.** `SOCKS5_SIGNATURE_AUTH=true` is the
   default and, as of `0.3.1`, the server refuses to start without a
   provisioned account. Do not set this to `false` on a public
   interface.

2. **Keep the tunnel on.** `SOCKS5_TUNNEL_ENCRYPTION=on`. Startup
   aborts if you try to run with signature auth on and the tunnel off.

3. **Provision the account before starting.** Verify
   `PeerManagement::get::<ProxyServer>("account")` returns an account
   on the host. If it does not, the server will exit — fix the
   provisioning, do not work around the guard.

4. **Set a low rate limit and quota.** `SOCKS5_RATE_MAX=5` and
   `SOCKS5_QUOTA_MAX=4` are reasonable defaults for a single-user
   proxy. Even with strict mode on, tightening these reduces the
   surface a scanner can consume while it fails authentication.

5. **Add a firewall allowlist** for known client IPs. Signature auth
   rejects unknown clients, but not letting them reach the listener at
   all is strictly better.

6. **Do not rely on RFC 1929 users on a public interface.** On the
   strict path the `SOCKS5_USERS` credentials are never reached; on
   the legacy path they are sent in the clear. Either way, the tunnel
   handshake is the authentication mechanism that matters.

Without a shared secret, an unauthenticated client cannot complete the
tunnel handshake and is dropped at the first byte. **As of `0.3.1` the
server no longer serves a plaintext SOCKS5 path in strict mode at
all** — the legacy path exists only when an operator explicitly sets
`SOCKS5_SIGNATURE_AUTH=false`, and only for clients that are known to
run on an already-protected transport.

### Post-incident checklist (2026-09-28)

If you are upgrading because of the 2026-09-28 incident:

1. Confirm `SOCKS5_SIGNATURE_AUTH=true` in the service environment.
2. Confirm an account is provisioned; the server will refuse to start
   otherwise.
3. Rotate the account's `ProxySecret` and re-issue client credentials.
   Assume any secret present on the wire during the incident is burned.
4. Add the source IPs from the incident logs to `nftables` /
   `fail2ban`. Strict mode will reject them, but keeping them off the
   listener removes them from the auth path entirely.
5. Packet-capture the listener port during normal operation and confirm
   the first server-side byte is `tunnel::MAGIC_WELCOME` and everything
   after is ciphertext — never an ASCII SOCKS5 greeting.

## Client

See [`CLIENT.md`](CLIENT.md). The client is a separate binary that
binds `127.0.0.1:1080`, completes the tunnel handshake with the server,
and forwards the browser's plaintext SOCKS5 conversation through the
encrypted channel. It performs no SOCKS5 parsing and no DNS
resolution.

## License

See repository.
