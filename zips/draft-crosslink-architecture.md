    ZIP: Unassigned
    Title: Crosslink Network Architecture
    Owners: Sam H. Smith <sam@shieldedlabs.net>
            Phillip Lujan <phillip@shieldedlabs.net>
            Mark Henderson <mark@shieldedlabs.net>
    Credits: Zooko Wilcox
             Daira-Emma Hopwood
             Nate Wilcox
             Andrew Mayerczyk
    Status: Draft
    Category: Consensus
    Created: 2026-04-09
    License: MIT
    Pull-Request: <https://github.com/zcash/zips/pull/???>


# Terminology

The key words "MUST", "REQUIRED", "MUST NOT", "SHOULD", and "MAY" in this
document are to be interpreted as described in BCP 14 [^BCP14] when, and
only when, they appear in all capitals.

The character &sect; is used when referring to sections of the Zcash Protocol Specification. [^protocol]

The terms "Mainnet" and "Testnet" are to be interpreted as described in &sect; 3.12 'Mainnet and Testnet'. [^protocol-networks]

The terms below are to be interpreted as follows:

Crosslink

: The hybrid Proof-of-Work / Proof-of-Stake consensus protocol for Zcash, combining the existing PoW chain (the "best chain" or "bc") with a BFT finality layer.

Best chain (bc)

: The Zcash Proof-of-Work blockchain, as defined by existing consensus rules (Equihash, difficulty adjustment, etc.), extended with a `fat_pointer_to_bft_block` field in the block header.

BFT chain

: The chain of BFT blocks produced by the Tenderlink consensus protocol. Each BFT block references exactly &sigma; consecutive PoW block headers.

Finality certificate (Fat Pointer)

: A `FatPointerToBftBlock` structure containing the BFT block hash, height, round, and a set of Ed25519 signatures from validators representing more than 2/3 of active stake. This cryptographically proves that a BFT block was committed.

Finalized block

: A PoW block whose height is at or below the `finalization_candidate_height` of a committed BFT block. Once finalized, a block MUST NOT be reorganized out of the best chain.

&sigma; (sigma, bc_confirmation_depth)

: The number of PoW block headers included in each BFT proposal. This determines the "confirmation depth" &mdash; the minimum number of PoW blocks that must exist atop a block before it can be finalized.

L (finalization_gap_bound)

: The maximum number of unfinalized PoW blocks before "Stalled Mode" activates. MUST be at least 2&sigma;.

Stalled Mode

: An emergency operating mode activated when the gap between the PoW tip height and the last finalized height exceeds L. In Stalled Mode, alternative consensus rules apply to maintain chain progress even if the BFT layer is unavailable.


# Abstract

This ZIP specifies the Crosslink network architecture: a hybrid Proof-of-Work / Proof-of-Stake consensus protocol for Zcash. Crosslink couples the existing PoW chain with a BFT finality layer (Tenderlink) to provide assured finality while preserving PoW mining.

This ZIP defines:

1. The extended PoW block header format (adding a finality certificate field).
2. The protocol parameters (&sigma; and L) and their semantics.
3. The coupling between the PoW chain and the BFT chain &mdash; how BFT blocks reference PoW headers and how finality propagates from BFT decisions to PoW blocks.
4. The structure and validation of finality certificates.
5. Stalled Mode behavior when the BFT layer is unavailable.
6. The finality status model for blocks and transactions.

The BFT consensus protocol (Tenderlink) is specified in [^zip-tenderlink]. The transport protocol (STP) is specified in [^zip-crosslink-transport]. The staking protocol is specified in [^zip-crosslink-staking].


# Motivation

Zcash currently provides only probabilistic finality. A transaction buried under $k$ blocks has an exponentially decreasing probability of being reversed, but this probability is never zero. This creates challenges for:

- **Bridges and interoperability.** Cross-chain protocols need a clear finality threshold to safely credit bridged assets.
- **Exchanges.** Confirmation requirements (e.g., 24 blocks) are heuristic and not formally guaranteed.
- **User experience.** Users cannot distinguish "very likely final" from "definitely final."

Crosslink provides *assured finality*: once a block is finalized, it cannot be reversed unless more than 1/3 of validator stake is Byzantine. This is a fundamentally stronger guarantee than probabilistic finality, backed by economic security (staked ZEC at risk of slashing).

The hybrid design preserves PoW mining, maintaining Zcash's existing security properties, miner participation, and ZEC distribution mechanism, while layering assured finality on top.


# Privacy Implications

Crosslink introduces finality certificates into PoW block headers. These certificates contain the Ed25519 public keys and signatures of the validators who committed the corresponding BFT block. This reveals:

1. Which validators are active at each height.
2. Which specific validators participated in each finality decision.

This information is inherent to any BFT-based finality protocol and is already partially visible through on-chain staking transactions. The privacy impact is limited to validator operations; user transaction privacy (shielded pools, etc.) is unaffected.


# Requirements

1. PoW blocks that have been finalized MUST NOT be reorganized.
2. The PoW chain MUST continue to function (with reduced guarantees) if the BFT layer stalls.
3. Finality certificates MUST be efficiently verifiable by any node (including light clients).
4. The coupling between PoW and BFT MUST be minimal: the BFT layer treats PoW headers as opaque payloads.
5. The protocol MUST be parameterized by &sigma; (confirmation depth) and L (finalization gap bound).


# Non-requirements

1. **Instant finality.** Finality lags behind the PoW tip by at least &sigma; blocks. This is a deliberate trade-off for security.
2. **PoW chain modification.** Existing PoW consensus rules (difficulty adjustment, Equihash, transaction validation) are unchanged except for the block header extension.


# Specification

## Protocol Parameters

| Parameter | Symbol | Prototype Value | Description |
|-----------|--------|-----------------|-------------|
| Confirmation depth | &sigma; | 3 | PoW headers per BFT proposal |
| Finalization gap bound | L | 7 | Stalled Mode threshold |

These parameters are consensus constants defined at activation and MUST NOT change without a network upgrade.

**Constraint:** L MUST be at least 2&sigma;.

## Extended Block Header

The PoW block header is extended with one new field appended after the existing fields:

| Existing fields | ... | (per current Zcash protocol specification) |
|-----------------|-----|---------------------------------------------|
| **New field** | **Type** | **Description** |
| fat_pointer_to_bft_block | FatPointerToBftBlock | Finality certificate referencing the most recent committed BFT block |

This field is appended to the header serialization after the Equihash solution. For blocks before the Crosslink activation height, this field MUST be the null fat pointer (44 zero bytes, zero signatures).

## Finality Certificate (FatPointerToBftBlock)

A finality certificate proves that a BFT block was committed by more than 2/3 of validator stake.

### Structure

| Field | Type | Size | Description |
|-------|------|------|-------------|
| vote_data | [u8; 44] | 44 | Vote data without voter identity (see below) |
| signature_count | u16 LE | 2 | Number of validator signatures |
| signatures | FatPointerSignature[] | 96 &times; N | Array of (public_key, signature) pairs |

### Vote Data Layout (44 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 32 | block_hash | BLAKE3 hash of the committed BFT block |
| 32 | 8 | height | BFT height (u64 LE) |
| 40 | 4 | round_commit | Round number with MSB set (bit 31 = 1 indicates precommit) |

### Signature Entry (96 bytes)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 32 | Ed25519 public key of the validator |
| 32 | 64 | Ed25519 signature over the 76-byte vote (see below) |

### Signed Data

Each signature in the finality certificate is an Ed25519 signature over a 76-byte vote:

| Offset | Size | Field |
|--------|------|-------|
| 0 | 32 | Validator's Ed25519 public key |
| 32 | 32 | BLAKE3 hash of the BFT block (from vote_data[0..32]) |
| 64 | 8 | BFT height (from vote_data[32..40]) |
| 72 | 4 | Round | commit flag (from vote_data[40..44]) |

The signature covers the concatenation of the validator's own public key with the 44-byte vote_data.

### Null Fat Pointer

A null finality certificate consists of 44 zero bytes and zero signatures. It is used:

- In PoW blocks before Crosslink activation.
- In the genesis BFT block's `previous_block_fat_ptr`.
- When no BFT decision has occurred since the last PoW block.

### Validation

A finality certificate is valid if and only if:

1. `signature_count` is greater than zero (unless null).
2. Every Ed25519 signature verifies against its corresponding public key and the 76-byte vote.
3. The signing validators are members of the active roster at the certificate's BFT height.
4. The total stake of the signing validators exceeds the big threshold (2f+1 of total active stake).

Implementations SHOULD use Ed25519 batch verification [^ed25519-batch] for efficiency.

### Serialization

```
vote_data:          44 bytes
signature_count:    2 bytes (u16 LE)
signatures:         96 * signature_count bytes
                    (each: 32 bytes pubkey || 64 bytes signature)
```

Total size: 46 + 96N bytes, where N is the number of signing validators.

## PoW-BFT Coupling

### BFT Block Construction

When a BFT proposer constructs a new proposal:

1. Start from the current PoW best chain tip.
2. Traverse backward through `previous_block_hash` pointers, collecting exactly &sigma; block headers (tip first, then parents).
3. The headers are stored deepest-first in the BFT block (the finalization candidate is `headers[0]`).
4. The `finalization_candidate_height` is the height of `headers[0]` &mdash; the deepest (oldest) header in the set.

### Finalization

When a BFT block is decided (receives 2f+1 precommits):

1. The PoW block at `finalization_candidate_height` becomes **finalized**.
2. All PoW blocks at heights &le; `finalization_candidate_height` that are in the best chain are also finalized (finality is monotonic).
3. The finality certificate is embedded in subsequent PoW blocks via the `fat_pointer_to_bft_block` header field.

### Fat Pointer Monotonicity

The `fat_pointer_to_bft_block` in a PoW block header MUST reference a BFT block at a height greater than or equal to the BFT block referenced by the parent PoW block's fat pointer. That is, fat pointers MUST NOT regress.

A PoW block that violates this rule MUST be rejected as invalid.

## Finality Status Model

Every block in the node's state has one of three finality statuses:

| Status | Condition | Meaning |
|--------|-----------|---------|
| NotYetFinalized | height > finalized_height | May or may not be finalized in the future |
| Finalized | height &le; finalized_height AND in best chain | Permanently part of the canonical chain |
| CantBeFinalized | height &le; finalized_height AND NOT in best chain | Permanently excluded (orphaned below the finality line) |

A transaction inherits the finality status of the block that contains it.

Nodes MUST expose the finality status of blocks and transactions through their RPC interfaces.

## Stalled Mode

If the gap between the PoW tip height and the last finalized height exceeds L:

$$\text{tip\_height} - \text{finalized\_height} > L$$

the network enters **Stalled Mode**.

In Stalled Mode:

1. The PoW chain continues to operate under its existing consensus rules.
2. The BFT layer continues attempting to reach consensus.
3. Nodes MUST NOT treat the absence of recent finality certificates as grounds to reject otherwise-valid PoW blocks.
4. Once the BFT layer resumes and finalizes a block, Stalled Mode ends.

The purpose of Stalled Mode is to ensure the PoW chain remains live even if the BFT validator set is partitioned, offline, or unable to reach consensus. This preserves Zcash's liveness guarantee: the chain always makes progress, even if finality is temporarily unavailable.

> [!note]
> The precise consensus rule changes during Stalled Mode (e.g., whether the fat pointer field is relaxed, whether fork choice changes) are under active design. This section will be updated as the design matures.

## Activation

Crosslink activates at a consensus-defined block height. At activation:

1. The PoW block at the activation height includes the first non-null `fat_pointer_to_bft_block`.
2. The initial BFT height is 0.
3. The initial roster is determined by staking transactions accumulated before activation.
4. The initial finalized height is the activation height itself.


# Rationale

**Fat pointers in PoW headers:** Embedding finality certificates directly in PoW block headers allows any node (including light clients) to verify finality without maintaining BFT chain state. The trade-off is increased header size, but this is bounded by the validator set size (at most 100 validators &times; 96 bytes = ~9.6 KB).

**Confirmation depth &sigma;:** Requiring &sigma; PoW headers per BFT proposal ensures that the finalization candidate has at least &sigma; blocks of PoW work atop it. This provides a security margin against short-range PoW attacks during the BFT voting window.

**Monotonic finality:** Once a block is finalized, it is never un-finalized. This is enforced by the fat pointer monotonicity rule and provides the strong guarantee that bridges and applications require.

**Stalled Mode:** A pure PoS system halts if >1/3 of validators go offline. Crosslink's hybrid design avoids this: the PoW chain continues regardless. Stalled Mode explicitly encodes this design choice, trading temporary loss of finality for permanent liveness.

**BLAKE3 for value IDs and block hashing:** BLAKE3 provides fast, keyed hashing with a simple API. Keyed mode (via `derive_key`) provides domain separation between different uses (proposer selection, value IDs, connection contention) without the complexity of hash function negotiation.

**Ed25519 for validator signatures:** Ed25519 provides fast signing and verification with small keys and signatures. Batch verification further improves performance for validating finality certificates with many signatures. The choice to use individual signatures (rather than aggregate signatures) simplifies the prototype; signature aggregation is a planned optimization.


# Deployment

Crosslink is deployed as a network upgrade (NU). The activation height is determined by the Zcash community governance process.

Prior to activation, validators MUST:

1. Register and stake ZEC via the staking protocol [^zip-crosslink-staking].
2. Establish STP connections with peers [^zip-crosslink-transport].
3. Run a Tenderlink consensus node [^zip-tenderlink].

The activation height MUST be far enough in the future to allow a sufficient validator set to form.


# Reference implementation

- [crosslink_monolith/librustzcash/zcash_primitives/src/bft.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; BftBlock, FatPointerToBftBlock, Vote, Blake3Hash
- [crosslink_monolith/zebra-crosslink/zebra-chain/src/block/header.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; Extended PoW block header with fat_pointer_to_bft_block field
- [crosslink_monolith/zebra-crosslink/zebra-state/src/crosslink.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; TFLBlockFinality enum, TFLService request/response types
- [crosslink_monolith/zebra-crosslink/zebra-crosslink/src/lib.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; TFLServiceInternal, PoW-BFT integration


# Open issues

1. **Stalled Mode rules.** The precise consensus rule changes during Stalled Mode are under active design. Key questions include: Is the fat pointer field relaxed? Does fork choice change? How is finality restored after a stall?
2. **Fat pointer size.** With 100 validators, a finality certificate is approximately 9.6 KB. This significantly increases block header size. Signature aggregation (BLS or Schnorr) could reduce this to a constant ~100 bytes.
3. **Light client verification.** Light clients can verify finality certificates, but they need the validator roster to check stake thresholds. A mechanism for light clients to obtain and verify rosters needs to be specified.
4. **Dynamic &sigma; and L.** The prototype uses fixed values. It may be desirable to adjust these parameters based on network conditions (e.g., increase &sigma; if hashrate drops).
5. **Cross-chain awareness.** Both the PoW and BFT chains reference each other. The BFT chain contains PoW headers; PoW blocks contain fat pointers. The interaction between PoW reorganizations and BFT finality needs formal safety analysis.
6. **Security analysis.** The interplay between PoW security (hashrate-based) and PoS security (stake-based) under various attack models (long-range, grinding, nothing-at-stake) requires rigorous formal analysis.


# References

[^BCP14]: [Information on BCP 14 &mdash; "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^protocol-networks]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1]. Section 3.12: Mainnet and Testnet](protocol/protocol.pdf#networks)

[^tfl-book]: [Zcash Trailing Finality Layer &sect;3.3 The Crosslink Construction](https://electric-coin-company.github.io/tfl-book/design/crosslink.html)

[^ed25519-batch]: [Hisil, Wong, Carter, Dawson. "Twisted Edwards Curves Revisited." ASIACRYPT 2008.](https://eprint.iacr.org/2008/522)

[^zip-tenderlink]: [ZIP-XXX: Tenderlink: BFT Consensus Protocol for Crosslink](draft-crosslink-tenderlink)

[^zip-crosslink-transport]: [ZIP-XXX: Crosslink Transport Protocol (STP)](draft-crosslink-transport)

[^zip-crosslink-staking]: [ZIP-XXX: Crosslink Proof-of-Stake and Validator Staking](draft-crosslink-staking)
