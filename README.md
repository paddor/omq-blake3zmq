# BLAKE3ZMQ

> [!WARNING]
> This is experimental and the gem is not maintained by cryptographers.
> It has not been independently audited. For production, use CurveZMQ instead.

BLAKE3ZMQ is a security mechanism for [OMQ](https://github.com/paddor/omq)
that replaces [CurveZMQ](https://rfc.zeromq.org/spec/26/) with modern
primitives:

* **X25519** key exchange (perfect forward secrecy)
* **ChaCha20-BLAKE3** AEAD (32-byte authentication tags)
* **BLAKE3** transcript hash and key derivation

It implements the `Protocol::ZMTP::Mechanism::Blake3` class for use with
[protocol-zmtp](https://github.com/paddor/protocol-zmtp).

See [RFC.md](RFC.md) for the full protocol specification.

## Features

* 4-message handshake (HELLO, WELCOME, INITIATE, READY)
* Transcript hash binding across all handshake messages; all post-handshake frames (including commands) AEAD-encrypted, unlike CurveZMQ
* Deterministic nonces (no per-message randomness needed)
* 32 bytes overhead per message (no wire nonce, no command wrapper)
* Stateless server until client authentication (cookie mechanism)
* Mutual authentication or server-only (anonymous client) modes
* Works out of the box (native C X25519 + Rust AEAD included)
* Crypto-backend-agnostic (can substitute your own primitives)

## Installation

```ruby
gem "omq-blake3zmq"
```

Batteries included: ships with [x25519](https://github.com/RubyCrypto/x25519)
(native C) and [chacha20blake3](https://github.com/paddor/chacha20blake3)
(Rust native).

## Usage

```ruby
require "omq"
require "omq/blake3zmq"

Crypto = OMQ::Blake3ZMQ::Crypto

# Generate or load keys
server_sk = Crypto::PrivateKey.generate
server_pk = server_sk.public_key.to_s

client_sk = Crypto::PrivateKey.generate
client_pk = client_sk.public_key.to_s

# Server socket
server = OMQ::REP.new
server.mechanism = Protocol::ZMTP::Mechanism::Blake3.server(
  public_key: server_pk,
  secret_key: server_sk.to_s,
  authenticator: ->(peer) { peer.public_key.to_s == client_pk },
)
server.bind("tcp://127.0.0.1:9999")

# Client socket (keys optional; omit for anonymous/ephemeral identity)
client = OMQ::REQ.new
client.mechanism = Protocol::ZMTP::Mechanism::Blake3.client(
  server_key: server_pk,
  public_key: client_pk,
  secret_key: client_sk.to_s,
)
client.connect("tcp://127.0.0.1:9999")
```

## Wire format

### Handshake

Four ZMTP command frames over plain TCP:

```
Client                              Server
  |                                    |
  |-- HELLO (C', hello_box) ---------> |  232 bytes
  |                                    |
  | <-- WELCOME (welcome_box) -------- |  224 bytes
  |                                    |
  |-- INITIATE (cookie, init_box) ---> |  variable
  |                                    |
  | <-- READY (ready_box) ------------ |  variable
  |                                    |
  |====== encrypted data phase ========|
```

HELLO carries the client's ephemeral public key `C'` and a box proving
knowledge of the server's permanent public key `S`. WELCOME returns the
server's ephemeral key `S'` inside a stateless cookie. INITIATE sends
back the cookie plus the client's permanent key `C` and a vouch binding
`C'` to `C`. READY confirms the session.

### Data phase

Every post-handshake frame (data and command) is AEAD-encrypted:

```
+-----------+----------------+------------------+-----------+
| flags     | length         | ciphertext       | tag       |
| (1 byte)  | (1 or 8 bytes) | (N bytes)        | (32 bytes)|
+-----------+----------------+------------------+-----------+
```

The `flags` byte and `length` bytes are authenticated as AAD. No counter
is sent on the wire; both peers derive per-message nonces from a
synchronized monotonic counter and a session-unique nonce prefix.

Unlike CurveZMQ, SUBSCRIBE/CANCEL/JOIN/LEAVE/PING/PONG/ERROR are all
AEAD-encrypted. CurveZMQ sends these in plaintext, leaking subscription
topics, group memberships, and heartbeat content.

## Constants

| Constant | Value |
|---|---|
| Key size | 32 bytes |
| Nonce size | 24 bytes |
| Tag size | 32 bytes |
| Cookie size | 152 bytes (24 nonce + 96 payload + 32 tag) |
| Mechanism name | `BLAKE3` (6 octets, null-padded to 20) |
| Protocol ID | `BLAKE3ZMQ-1.0` |
| HELLO body | 232 bytes |
| WELCOME body | 224 bytes |

## Per-message overhead

| | BLAKE3ZMQ | CurveZMQ |
|---|---:|---:|
| Tag size | 32 bytes | 16 bytes |
| Counter on wire | 0 bytes | 8 bytes |
| Command wrapper | none | 17 bytes |
| **Total** | **32 bytes** | **41 bytes** |

BLAKE3ZMQ has 9 bytes less overhead per message despite a larger tag.

## Benchmarks

CurveZMQ (RbNaCl/libsodium) vs BLAKE3ZMQ (Rust native ChaCha20-BLAKE3 + C native X25519).

Ruby 4.0.2, x86_64 Linux.

### Handshake latency (100 rounds)

| | Time | Per handshake |
|---|---:|---:|
| CurveZMQ/RbNaCl | 226 ms | 2.26 ms |
| BLAKE3ZMQ | 120 ms | 1.20 ms |
| **Speedup** | | **1.9x** |

### Message encrypt + decrypt throughput (20,000 messages)

| Size | CurveZMQ/RbNaCl | BLAKE3ZMQ | Speedup |
|---:|---:|---:|---:|
| 64 B | 8.4 MB/s | 16.1 MB/s | 1.9x |
| 256 B | 38.0 MB/s | 51.4 MB/s | 1.4x |
| 1 KB | 88.1 MB/s | 137.5 MB/s | 1.6x |
| 4 KB | 196.3 MB/s | 265.5 MB/s | 1.4x |
| 16 KB | 289.1 MB/s | 387.1 MB/s | 1.3x |
| 64 KB | 413.3 MB/s | 452.4 MB/s | 1.1x |
| 128 KB | 426.4 MB/s | 527.0 MB/s | 1.2x |
| 256 KB | 428.7 MB/s | 538.0 MB/s | 1.3x |

### Full round-trip (handshake + 1,000 messages over UNIXSocket)

| Size | CurveZMQ/RbNaCl | BLAKE3ZMQ | Speedup |
|---:|---:|---:|---:|
| 64 B | 69.4 ms (0.9 MB/s) | 50.8 ms (1.3 MB/s) | 1.4x |
| 1 KB | 54.6 ms (18.7 MB/s) | 62.6 ms (16.3 MB/s) | 0.9x |
| 64 KB | 281.7 ms (232.6 MB/s) | 242.2 ms (270.6 MB/s) | 1.2x |

Run benchmarks yourself:

```
OMQ_DEV=1 bundle exec ruby bench/throughput.rb
```

## Development

```
bundle install
bundle exec rake test
```

## License

[ISC](LICENSE)
