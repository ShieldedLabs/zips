    ZIP: Unassigned
    Title: Crosslink Transport Protocol (STP)
    Owners: Sam H. Smith <sam@shieldedlabs.net>
            Phillip Lujan <phillip@shieldedlabs.net>
            Mark Henderson <mark@shieldedlabs.net>
    Credits: Andrew Mayerczyk
    Status: Draft
    Category: Network
    Created: 2026-04-09
    License: MIT
    Pull-Request: <https://github.com/zcash/zips/pull/???>


# Terminology

The key words "MUST", "REQUIRED", "MUST NOT", "SHOULD", and "MAY" in this
document are to be interpreted as described in BCP 14 [^BCP14] when, and
only when, they appear in all capitals.

The character &sect; is used when referring to sections of the Zcash Protocol Specification. [^protocol]

The terms below are to be interpreted as follows:

STP

: Secure Transport Protocol. The UDP-based encrypted transport layer defined by this ZIP.

Packlet

: The smallest unit of framing within an STP packet. One UDP datagram may carry one or more packlets.

Jumbogram

: A logical message that exceeds the path MTU and is fragmented across multiple STP packets for reassembly at the receiver.

Connection Key

: A tuple of (IPv6 address, port, 15-bit public key fingerprint) that uniquely identifies an STP peer.

BFT Validator

: A Crosslink proof-of-stake validator participating in the Tenderlink BFT consensus protocol. Validators use STP as their transport layer for consensus messages.


# Abstract

This ZIP specifies the Secure Transport Protocol (STP), a UDP-based peer-to-peer transport layer for the Zcash Crosslink hybrid consensus protocol. STP provides authenticated, encrypted communication between BFT validators using the Noise IK handshake pattern with Curve25519, ChaCha20-Poly1305, and BLAKE2s. The protocol supports fragmentation of large consensus messages (jumbograms), bandwidth estimation, and congestion-aware transmission.

STP is a general-purpose encrypted datagram transport, but this ZIP specifies it in the context of its use by the Tenderlink BFT consensus protocol (specified in a companion ZIP).


# Motivation

The existing Zcash peer-to-peer network uses TCP-based connections (inherited from Bitcoin) for block and transaction propagation. The Crosslink hybrid consensus protocol introduces a second consensus layer &mdash; a BFT finality protocol &mdash; that has fundamentally different networking requirements:

1. **Low latency.** BFT consensus is round-based with timeouts on the order of 10 seconds. TCP's head-of-line blocking and retransmission delays are unacceptable.
2. **Authenticated peers.** BFT validators are identified by Ed25519 public keys. Transport-layer authentication prevents impersonation without requiring application-layer signature checks on every message.
3. **Confidentiality.** Consensus votes reveal validator behavior. Encrypting the transport prevents passive observers from learning which validators voted for which proposals before finality is reached.
4. **Message-oriented framing.** BFT messages are discrete (proposals, prevotes, precommits), not streams. UDP datagrams are a natural fit.
5. **Large message support.** BFT proposals contain &sigma; PoW block headers, which may exceed the UDP path MTU. Fragmentation and reassembly at the transport layer avoids burdening the consensus layer with chunking logic.


# Privacy Implications

STP encrypts all consensus traffic between validators, preventing passive network observers from learning which validators are voting for which proposals. This provides a degree of vote privacy during the consensus round, though the finality certificate published on-chain reveals the final set of committing validators.

The Noise IK handshake pattern reveals the initiator's static public key to the responder but not to passive observers. The responder's static key is encrypted. This is acceptable because validator keys are public (they appear in the on-chain roster).


# Requirements

1. Authenticated, encrypted communication between BFT validators over UDP.
2. Resistance to replay attacks without requiring synchronized clocks.
3. Fragmentation and reassembly for messages up to 8 MB.
4. Congestion-aware transmission with bandwidth estimation.
5. Tolerance for packet reordering and loss without connection teardown.
6. Support for both IPv4 (as IPv6-mapped) and IPv6.


# Non-requirements

1. **Reliable delivery.** STP is best-effort. The BFT consensus protocol tolerates message loss through its round-based retry structure.
2. **Ordered delivery.** Messages may arrive out of order. The consensus layer handles ordering by height and round.
3. **NAT traversal.** Validators are expected to have publicly reachable IP addresses or to arrange port forwarding externally.


# Specification

## Wire Format

### UDP Datagram Layout

Every STP packet is a single UDP datagram with the following structure:

```
+--------+--------+-----------+------------------+-----------+
| Bytes  | 0-1    | 2-5       | 6..N-T           | N-T..N    |
+--------+--------+-----------+------------------+-----------+
| Field  | Prefix | Nonce     | Encrypted Payload | AEAD Tag |
+--------+--------+-----------+------------------+-----------+
```

Prefix (2 bytes)

: The first two bytes of the sender's STP public key, left-shifted by 1 bit. Used for demultiplexing incoming packets to the correct connection state without decryption.

Nonce (4 bytes, little-endian u32)

: A monotonically increasing virtual nonce. Used as the Noise nonce for AEAD decryption and for replay detection.

Encrypted Payload (variable)

: One or more packlets, encrypted under the connection's Noise transport state. For plaintext connections (testing only), this field is unencrypted.

AEAD Tag (16 bytes)

: The ChaCha20-Poly1305 authentication tag. Present only for encrypted connections. Absent for plaintext mode.

The minimum UDP datagram size for guaranteed delivery MUST be assumed to be 1208 bytes. Packets smaller than the path MTU inside the STP framing MUST be zero-padded to exactly the path MTU to prevent traffic analysis based on packet size.

### Path MTU Calculation

```
UDP_mMTU     = 1208 bytes  (minimum guaranteed UDP payload)
STP_OVERHEAD = 6 bytes (prefix + nonce) + crypto_overhead
             = 6 + 16 = 22 bytes (for Noise_IK)
PATH_MTU     = UDP_mMTU - STP_OVERHEAD = 1186 bytes
```

### Packlet Framing

Within the decrypted payload, data is organized as packlets. Each packlet begins with a 2-byte header:

```
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
|       Tag (2 bits)        |           Length (14 bits)           |
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
```

Tag values:

| Tag | Name               | Description                                            |
|-----|--------------------|--------------------------------------------------------|
| 0   | Acknowledgements   | ACK bitmap for reliable delivery feedback               |
| 1   | AnEntireDatagram   | A complete message that fits in a single packlet        |
| 2   | OneJumboFragment   | One fragment of a jumbogram (large message)             |
| 3   | ReliableStreamed   | Reserved for future use                                 |

Length: the number of bytes following the packlet header (maximum 16383).

### Jumbogram Fragment Header

When Tag = 2 (OneJumboFragment), the packlet payload begins with an 8-byte fragment descriptor:

```
+---+---+---+---+---+---+---+---+
| Jumbogram ID (18) | Total Length (23) | Byte Index (23) |
+---+---+---+---+---+---+---+---+
```

All fields are packed into a single little-endian u64:

- Bits 0&ndash;17: Jumbogram ID (18 bits, max 262,144 concurrent IDs)
- Bits 18&ndash;40: Total length of the complete message (23 bits, max 8,388,608 bytes)
- Bits 41&ndash;63: Byte offset where this fragment's data begins (23 bits)

The fragment data follows immediately after this 8-byte header.

**Reassembly rules:**

- A receiver MUST maintain at most 64 concurrent reassembly slots per connection.
- If a fragment arrives for a new jumbogram ID when 64 slots are occupied, the connection MUST be terminated.
- Overlapping fragments with identical content are idempotent and MUST be accepted.
- Overlapping fragments with differing content indicate corruption; the connection MUST be terminated.
- A message whose total length would fit in a single AnEntireDatagram packlet MUST NOT be sent as a jumbogram.

### Acknowledgement Packlet

When Tag = 0, the packlet payload contains:

```
+----------+----------+---------------------------+
| Serial   | Min ACK  | ACK Entries (3 bytes each)|
| (8 bytes)| (8 bytes)|                           |
+----------+----------+---------------------------+
```

- Serial (u64 LE): sender's packet serial number
- Min ACK (u64 LE): minimum acknowledged serial; MSB is ECN congestion flag
- ACK entries: each 3 bytes encoding a 23-bit offset from Min ACK and 1 ECN bit

## Connection Establishment

### Noise IK Handshake

STP uses the Noise IK handshake pattern [^noise]:

```
Noise_IK_25519_ChaChaPoly_BLAKE2s

  <- s
  ...
  -> e, es, s, ss
  <- e, ee, se
```

The initiator (client) knows the responder's static public key in advance (from the BFT roster or peer discovery).

**Client Hello:**

```
+-------------------+-------------------+
| Magic (6 bytes)   | Noise Message 1   |
+-------------------+-------------------+
```

Magic: the 6-byte constant `0x7193c304f8d5` (little-endian), identifying this as a `Noise_IK_25519_ChaChaPoly_BLAKE2s` connection. The magic is also used as the Noise prologue for channel binding.

The client retransmits Client Hello every 2.5 seconds until a Server Hello is received or 30 seconds have elapsed (connection timeout).

**Server Hello:**

```
+-------------------+-------------------+
| Magic (6 bytes)   | Noise Message 2   |
+-------------------+-------------------+
```

After Server Hello, both sides derive symmetric transport keys. All subsequent packets use the encrypted wire format described above.

**Dual-initiation tie-breaking:** If both peers simultaneously initiate connections to each other, the peer whose static public key is lexicographically smaller acts as client (initiator). The other peer acts as server (responder).

### Connection Maintenance

- **Keep-alive:** A connected peer MUST send an empty encrypted packet at least every 5 seconds if no other traffic has been sent.
- **Timeout:** A connection MUST be considered dead after 15 seconds without receiving any data.
- **Nonce tolerance:** A receiver MUST accept nonces up to 512 ahead of the last seen nonce (to tolerate packet reordering). Nonces more than 512 ahead or any nonce at or below the last seen nonce MUST be rejected.

## Bandwidth Estimation

STP estimates available bandwidth using RTT measurements and delivery rate tracking:

1. RTT is measured across 10 buckets, and the minimum is used as the base RTT.
2. Delivery rate is measured across 20 buckets, and the maximum is used as the bottleneck bandwidth.
3. The congestion window is computed as:
   ```
   cwnd = bottleneck_bandwidth * base_rtt
   ```
4. ECN markings from ACK packets are used to detect congestion without relying solely on loss.

The global maximum bandwidth per connection is 1,000,000 bytes per second.

## Peer Identity

Each STP peer is identified by an `STPAddress`:

| Field  | Size      | Description                                         |
|--------|-----------|-----------------------------------------------------|
| IP     | 16 bytes  | IPv6 address (IPv4 as ::ffff:0:0 mapped)            |
| Port   | 2 bytes   | UDP port number                                     |
| Magic  | 6 bytes   | Connection magic identifying the crypto suite        |
| Key    | 32 bytes  | Static public key (Curve25519)                       |

The `ConnectionKey` used for internal connection lookup is the tuple `(IP, Port, key_15_bits)` where `key_15_bits` is the lower 15 bits of the public key, providing fast demultiplexing.


# Rationale

**UDP over TCP:** BFT consensus messages are independent and loss-tolerant. TCP's reliability mechanisms (retransmission, flow control, ordered delivery) add latency without benefit. UDP allows the consensus layer to handle retransmission at the application level with knowledge of which messages are still relevant.

**Noise IK over TLS:** Noise IK provides mutual authentication in a single round trip with a simpler state machine than TLS 1.3. The IK pattern is appropriate because validator static keys are publicly known from the on-chain roster, so the initiator revealing its identity to the responder is not a privacy concern.

**Jumbograms over application-level chunking:** Handling fragmentation at the transport layer simplifies the BFT consensus implementation. The consensus layer sends and receives complete messages without managing chunk indices or reassembly state.

**Fixed-size padding:** Zero-padding all packets to the path MTU prevents traffic analysis that could distinguish proposals (large) from votes (small), preserving vote privacy during consensus rounds.

**BLAKE2s over BLAKE2b:** BLAKE2s is the Noise framework's standard hash for the `IK_25519_ChaChaPoly` suite. It is optimized for 32-bit platforms and produces 32-byte digests, matching the key size of Curve25519.


# Deployment

This protocol is deployed as part of the Crosslink network upgrade. STP is used exclusively by BFT validators participating in the Tenderlink consensus protocol. It does not replace or modify the existing Zcash P2P protocol used for block and transaction propagation.

STP listeners MUST bind to a separate UDP port from any existing Zcash services.


# Reference implementation

- [crosslink_monolith/tenderlink/src/bandwidth_test.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; STP connection state machine, Noise handshake, packlet framing, jumbogram reassembly
- [crosslink_monolith/tenderlink/src/native_sockets.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; UDP socket I/O, ECN/DSCP handling, platform-specific code


# Open issues

1. **Path MTU discovery.** The current implementation assumes a minimum MTU of 1208 bytes. Dynamic PMTU discovery would improve throughput on paths with larger MTUs.
2. **Peer discovery.** There is no built-in peer discovery protocol. Validators must be configured with peer addresses out of band (from the on-chain roster or a bootstrap list). A gossip-based discovery mechanism may be added in a future ZIP.
3. **Rekeying.** The implementation maintains a cipher triplet (old, current, new) for rekeying, but the rekeying schedule is not yet specified. A production deployment SHOULD rekey after a configurable number of bytes or elapsed time.
4. **ReliableStreamed tag.** Packlet tag 3 is reserved but not yet specified. This may be used for state synchronization (e.g., catching up a validator that was offline).
5. **Quantum resistance.** The current Noise suite uses Curve25519, which is not quantum-resistant. A future revision may adopt a hybrid or post-quantum key exchange.


# References

[^BCP14]: [Information on BCP 14 &mdash; "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^noise]: [The Noise Protocol Framework, Revision 34](https://noiseprotocol.org/noise.html)

[^tfl-book]: [Zcash Trailing Finality Layer, Electric Coin Company](https://electric-coin-company.github.io/tfl-book/)
