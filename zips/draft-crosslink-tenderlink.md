    ZIP: Unassigned
    Title: Tenderlink: BFT Consensus Protocol for Crosslink
    Owners: Sam H. Smith <sam@shieldedlabs.net>
            Phillip Lujan <phillip@shieldedlabs.net>
            Mark Henderson <mark@shieldedlabs.net>
    Credits: Andrew Mayerczyk
             Zooko Wilcox
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

The terms below are to be interpreted as follows:

Tenderlink

: The BFT (Byzantine Fault Tolerant) consensus protocol used within Zcash Crosslink. Derived from the Tendermint algorithm [^tendermint] with modifications for the hybrid PoW/PoS model.

Validator (Finalizer)

: A participant in the BFT consensus protocol, identified by an Ed25519 public key and weighted by staked ZEC. Validators vote on proposals and produce finality certificates.

Roster

: The ordered set of active validators for a given BFT height, sorted by public key. At most 100 validators are active at any height.

Proposal

: A BFT block containing exactly &sigma; (sigma) consecutive PoW block headers, submitted by the designated proposer for a given height and round.

Prevote

: The first voting phase. Validators prevote for a proposal's value ID (or nil) after validating it.

Precommit

: The second voting phase. Validators precommit for a value ID (or nil) after observing 2f+1 prevotes.

Value ID

: A BLAKE3 keyed hash of a serialized BFT block, computed using the key derived from `BLAKE3::derive_key("BFT Value ID")`. Used to identify proposals without transmitting full block data in votes.

f

: The maximum number of Byzantine-faulty stake units. For total active stake $N$: $f = \lfloor(N-1)/3\rfloor$.

Big threshold

: The stake required for consensus progress: $2f+1$. When $f=0$ (trivial case), the big threshold equals $N$.

Small threshold

: The stake required for round catch-up: $f+1$.


# Abstract

This ZIP specifies Tenderlink, the Byzantine Fault Tolerant consensus protocol used by Zcash Crosslink to provide finality for the Proof-of-Work chain. Tenderlink is a three-phase (Propose, Prevote, Precommit) round-based consensus algorithm derived from Tendermint. At each BFT height, validators agree on a set of &sigma; consecutive PoW block headers; when 2f+1 stake precommits to a proposal, the deepest PoW block in that set becomes finalized.

Tenderlink operates over the STP transport protocol specified in the companion ZIP [^zip-crosslink-transport] and produces finality certificates specified in [^zip-crosslink-architecture].


# Motivation

Zcash's Proof-of-Work chain provides probabilistic finality: the deeper a block is buried, the less likely a reorganization. This is sufficient for many use cases but insufficient for bridges, exchanges, and other systems that require assured finality.

Crosslink augments PoW with a BFT finality layer. Tenderlink provides the consensus mechanism for this layer. The key design goals are:

1. **Safety.** No two conflicting blocks are finalized unless more than 1/3 of validator stake is Byzantine.
2. **Liveness.** The protocol makes progress as long as more than 2/3 of validator stake is online and honest.
3. **Minimal coupling.** The BFT protocol treats PoW block headers as opaque payloads. The coupling between PoW and BFT is specified in [^zip-crosslink-architecture], not here.
4. **Deterministic proposer selection.** Proposer rotation is deterministic and weighted by stake, using BLAKE3-based hashing to prevent proposer prediction gaming.


# Privacy Implications

Validator votes (prevotes and precommits) are visible to all validators during consensus rounds. After finality, the set of precommitting validators is published in the finality certificate embedded in PoW blocks. This reveals which validators participated in finalizing each block.

This is inherent to BFT protocols and is mitigated by the transport-layer encryption specified in [^zip-crosslink-transport], which prevents passive observers from learning vote contents during the consensus round.


# Requirements

1. Agreement on a sequence of BFT blocks, each containing exactly &sigma; PoW headers.
2. Tolerance of up to f Byzantine validators (where f = &lfloor;(N-1)/3&rfloor; of total active stake).
3. Deterministic, stake-weighted proposer selection.
4. Round-based timeout escalation for liveness under partial participation.
5. Locking mechanism to prevent validators from equivocating across rounds.


# Non-requirements

1. **Transaction ordering.** Tenderlink does not order transactions; it finalizes PoW blocks that contain transactions.
2. **Validator set changes.** The roster is determined by the staking protocol specified in [^zip-crosslink-staking]. Tenderlink receives the roster as input after each height decision.
3. **Slashing.** Equivocation detection is performed by Tenderlink (and evidence is available in the form of conflicting signed votes), but the slashing mechanism itself is specified in [^zip-crosslink-staking].


# Specification

## Protocol Parameters

| Parameter | Symbol | Prototype Value | Description |
|-----------|--------|-----------------|-------------|
| Confirmation depth | &sigma; | 3 | Number of PoW headers per BFT proposal |
| Max active validators | &mdash; | 100 | Maximum validators considered in any round |
| Propose timeout base | &mdash; | 10,000 ms | Base timeout for the Propose phase |
| Prevote timeout base | &mdash; | 10,000 ms | Base timeout for the Prevote phase |
| Precommit timeout base | &mdash; | 10,000 ms | Base timeout for the Precommit phase |
| Timeout increment | &mdash; | 2,500 ms per round | Linear backoff per round for all phases |

The timeout for phase $P$ in round $r$ is:

$$T_P(r) = 10000\text{ms} + r \times 2500\text{ms}$$

## BFT Block Structure

A BFT block (the proposal value) is serialized as:

| Field | Type | Size | Description |
|-------|------|------|-------------|
| version | u32 LE | 4 | Protocol version (currently 1) |
| height | u32 LE | 4 | BFT chain height |
| previous_block_fat_ptr | FatPointer | variable | Finality certificate for the previous BFT block |
| finalization_candidate_height | u32 LE | 4 | PoW height of the block being finalized |
| header_count | u32 LE | 4 | Number of PoW headers (MUST equal &sigma;) |
| headers | BcBlockHeader[] | variable | Exactly &sigma; PoW block headers, deepest first |

The **Value ID** of a BFT block is `BLAKE3_keyed(key, serialize(block))` where `key = BLAKE3::derive_key("BFT Value ID")`.

### Validation

A BFT block proposal MUST satisfy all of the following:

1. `header_count` equals &sigma;.
2. Each header's `previous_block_hash` correctly chains to the next header (headers are ordered deepest first).
3. Each header's Equihash solution is valid.
4. The version field is a known, expected value.

Additionally, the following **stateful** validations are performed by the node but are not part of the BFT block's intrinsic validity:

1. The difficulty thresholds are within correct bounds for the Difficulty Adjustment Algorithm.
2. The timestamps are within correct bounds.
3. The headers are consistent with the node's view of the PoW best chain.

## Consensus Algorithm

### State Variables

Each validator maintains:

| Variable | Description |
|----------|-------------|
| height | Current BFT height being decided |
| round | Current round number (0-indexed) |
| step | Current phase: Propose, Prevote, or Precommit |
| locked_value, locked_round | Value and round of the last precommitted value with 2f+1 prevotes. Once locked, the validator MUST NOT prevote or propose a different value unless unlocked by a higher round. |
| valid_value, valid_round | Most recently validated value with 2f+1 prevotes. Used for re-proposal. |

### Proposer Selection

The proposer for a given (height, round) is determined by:

```
key   = BLAKE3::derive_key("BFT Proposer")
hash  = BLAKE3_keyed(key, height_le_bytes || round_le_bytes)
stake = u64_from_le(hash[0..8]) % total_active_roster_stake
index = min i such that roster[i].cumulative_stake > stake
```

where `cumulative_stake` is the running sum of stake for validators 0 through i (inclusive), and the roster is sorted by public key.

This produces a deterministic, stake-weighted proposer assignment.

### Round Lifecycle

Each round proceeds through three phases:

#### Phase 1: Propose

The designated proposer broadcasts a proposal:

- If `valid_value` is set (from a previous round), the proposer MUST re-propose `valid_value`.
- Otherwise, the proposer constructs a new BFT block from the top &sigma; PoW headers and broadcasts it.
- All validators start a Propose timeout. If no valid proposal is received before timeout, the validator prevotes nil.

Proposals are transmitted as signed chunks. Each chunk contains a `PacketProposalChunkHeader`:

| Field | Type | Size | Description |
|-------|------|------|-------------|
| chunk_i | u32 LE | 4 | Chunk index |
| proposal_size | u32 LE | 4 | Total proposal size in bytes |
| round | u32 LE | 4 | Round number |
| valid_round | u32 LE | 4 | Round of valid_value (-1 encoded as 0xFFFFFFFF) |
| height | u64 LE | 8 | BFT height |
| proposal_id | [u8; 32] | 32 | Value ID of the complete proposal |

Total: 56 bytes. Each chunk is individually signed by the proposer with Ed25519.

#### Phase 2: Prevote

A validator prevotes for a value ID or nil based on the following rules:

**Rule 22 (first proposal, valid_round = -1):** Upon receiving a valid proposal with `valid_round = -1` while in step Propose:

- If the proposal is valid AND (`locked_round = -1` OR `locked_value = proposal`): prevote for the proposal's value ID.
- Otherwise: prevote nil.

**Rule 28 (re-proposed value):** Upon receiving a valid proposal with `0 <= valid_round < round` AND 2f+1 prevotes for the value ID at `valid_round`, while in step Propose:

- If the proposal is valid AND (`locked_round <= valid_round` OR `locked_value = proposal`): prevote for the proposal's value ID.
- Otherwise: prevote nil.

**Timeout:** If the Propose timeout fires while still in step Propose: prevote nil.

#### Phase 3: Precommit

**Rule 36 (lock and precommit):** Upon receiving a valid proposal AND 2f+1 prevotes for its value ID, while in step Prevote:

- Set `locked_value = proposal`, `locked_round = round`.
- Precommit for the value ID.
- Set `valid_value = proposal`, `valid_round = round`.

**Rule 44 (nil precommit):** Upon 2f+1 nil prevotes, while in step Prevote: precommit nil.

**Rule 34 (start precommit timeout):** Upon 2f+1 prevotes (any value, including nil), while in step Prevote, for the first time: start the Prevote timeout.

**Rule 47 (start decision timeout):** Upon 2f+1 precommits (any value), for the first time: start the Precommit timeout.

#### Decision

**Rule 49 (value decided):** Upon receiving a valid proposal AND 2f+1 precommits for its value ID (at any round for the current height):

1. The BFT block is decided.
2. Construct a finality certificate (FatPointerToBftBlock) from the precommit signatures.
3. The PoW block at `finalization_candidate_height` becomes finalized.
4. Advance to the next height. Reset `locked_value`, `locked_round`, `valid_value`, `valid_round` to nil/-1.
5. Request the updated roster from the staking protocol.
6. Begin round 0 of the new height.

#### Round Catch-up

**Rule 55:** Upon receiving f+1 messages (any type) for a round higher than the validator's current round at the current height: skip to that round immediately (call `start_round`).

#### Timeout Behavior

| Timeout fires | While in step | Action |
|---------------|---------------|--------|
| Propose       | Propose       | Prevote nil |
| Prevote       | Prevote       | Precommit nil |
| Precommit     | Any           | Start round + 1 |

## Vote Serialization

A vote is a 76-byte structure:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 32 | validator_address | Ed25519 public key of the voter |
| 32 | 32 | value | BLAKE3 value ID (or all zeros for nil) |
| 64 | 8 | height | BFT height (u64 LE) |
| 72 | 4 | round_and_type | Round (bits 0-30) | is_precommit flag (bit 31) |

A **signed vote** appends a 64-byte Ed25519 signature of the 76-byte vote, for a total of 140 bytes.

### Wire Format for Vote Packets

Votes are batched into `PacketVotes` structures for network efficiency:

| Field | Type | Size | Description |
|-------|------|------|-------------|
| no_votes_n | u8 | 1 | Number of nil votes |
| yes_votes_n | u8 | 1 | Number of non-nil votes |
| round | u32 LE | 4 | Round |
| height | u64 LE | 8 | BFT height |
| value_id | [u8; 32] | 32 | The non-nil value ID |
| votes | PubKeySig[18] | variable | Array of (roster_index: u16, signature: [u8; 64]) |

Maximum packet size: 1240 bytes (fits within one STP packlet at minimum MTU).

Nil votes appear first in the votes array, followed by yes votes. All yes votes are for the same `value_id`.

## Message Types

| Packet Type | Value | Description |
|-------------|-------|-------------|
| PROPOSAL_CHUNK | 7 | One chunk of a proposal, signed by proposer |
| PREVOTE_SIGNATURES | 8 | Batched prevote signatures |
| PRECOMMIT_SIGNATURES | 9 | Batched precommit signatures |

Each message is prefixed with a 16-byte packet header (8-byte tag + 8-byte reserved) and optionally a 28-byte status field.

The status flag (bit 7 of the tag byte) indicates that a `PacketStatus` is appended after the packet header, enabling peers to piggyback synchronization requests on consensus messages.

### PacketStatus

| Field | Type | Size | Description |
|-------|------|------|-------------|
| height | u64 LE | 8 | Height of interest |
| round | u32 LE | 4 | Round of interest |
| need_proposal_chunk_ranges | [u32; 2] | 8 | Range of needed proposal chunks [lo, hi) |
| need_prevote_ranges | [u16; 2] | 4 | Range of needed prevote indices |
| need_precommit_ranges | [u16; 2] | 4 | Range of needed precommit indices |

Total: 28 bytes.

## Equivocation Detection

A validator is considered **equivocating** (Byzantine) if it:

1. Proposes two different values in the same (height, round): detected by observing two proposal chunks with different `proposal_id` or `proposal_size` values for the same proposer, height, and round.
2. Casts two different votes (prevote or precommit) in the same (height, round): detected by observing two signed votes from the same validator with different value IDs.

Upon detecting equivocation, the node:

1. MUST log the evidence (both conflicting signed messages).
2. MUST ignore the second message.
3. MAY submit slashing evidence to the staking protocol.

### Amnesiac Proposer

A special case exists where a validator proposes a different value after restarting (amnesia). If a validator observes its *own* conflicting proposal, it MUST flush its local proposal state and adopt the latest proposal rather than marking itself as faulty. This is the "Amnesiac Proposer's Dilemma" &mdash; a validator that has been disconnected and reconnected may encounter its own prior proposals from before the disconnect.


# Rationale

**Tendermint-derived:** Tenderlink is based on the well-studied Tendermint BFT algorithm, which provides optimal Byzantine fault tolerance (tolerating up to f < N/3 faulty validators) with a simple three-phase structure. The modifications for Crosslink are minimal: proposals contain PoW headers instead of transaction batches, and proposer selection uses BLAKE3 instead of a round-robin scheme.

**Stake-weighted proposer selection via hashing:** Using `BLAKE3_keyed(height || round) % total_stake` and binary search on cumulative stake provides uniform-per-stake proposer probability without maintaining round-robin state across restarts. This is robust to validator set changes between heights.

**Chunked proposals:** BFT proposals containing &sigma; full PoW block headers (each ~1487 bytes with Equihash solutions) may exceed the STP path MTU. Chunking at the consensus layer (in addition to STP jumbograms) allows each chunk to be individually signed by the proposer, enabling incremental validation and early rejection of invalid proposals.

**18-vote batching:** The PacketVotes structure fits exactly 18 votes (each 66 bytes: 2-byte roster index + 64-byte signature) within the 1240-byte packet size limit, maximizing network efficiency.

**Linear timeout backoff:** Increasing timeouts by 2500ms per round provides graceful degradation under network congestion or partial validator availability, while keeping initial rounds responsive.


# Deployment

Tenderlink is activated as part of the Crosslink network upgrade at a consensus-defined activation height. Before activation, validators MUST have:

1. Registered and staked via the staking protocol.
2. Established STP connections with a sufficient set of peers.
3. Synchronized their view of the PoW chain to the current tip.


# Reference implementation

- [crosslink_monolith/tenderlink/src/lib.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; Tenderlink BFT consensus engine
- [crosslink_monolith/librustzcash/zcash_primitives/src/bft.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; Core data structures (BftBlock, Vote, FatPointer)


# Open issues

1. **Timeout tuning.** The prototype timeout values (10s base, 2.5s increment) are conservative. Production values should be tuned based on observed network latency and block propagation times.
2. **View change optimization.** The current round catch-up (Rule 55) triggers on f+1 messages. More sophisticated view change protocols could reduce latency.
3. **Proposal pipelining.** The current protocol proposes only after the previous height is decided. Pipelining proposals could improve throughput at the cost of complexity.
4. **Signature aggregation.** Individual Ed25519 signatures per validator are bandwidth-intensive. BLS aggregate signatures or Schnorr multisignatures could reduce finality certificate size. This is tracked as a potential productionization improvement.
5. **Catch-up protocol.** Validators that fall behind (offline for multiple heights) need a catch-up mechanism to sync BFT state. The current implementation stores recent commit round data but does not specify a formal catch-up wire protocol.


# References

[^BCP14]: [Information on BCP 14 &mdash; "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^tendermint]: [Buchman, Kwon, Milosevic. "The latest gossip on BFT consensus." arXiv:1807.04938, 2018.](https://arxiv.org/abs/1807.04938)

[^tfl-book]: [Zcash Trailing Finality Layer, Electric Coin Company](https://electric-coin-company.github.io/tfl-book/)

[^zip-crosslink-transport]: [ZIP-XXX: Crosslink Transport Protocol (STP)](draft-crosslink-transport)

[^zip-crosslink-architecture]: [ZIP-XXX: Crosslink Network Architecture](draft-crosslink-architecture)

[^zip-crosslink-staking]: [ZIP-XXX: Crosslink Proof-of-Stake and Validator Staking](draft-crosslink-staking)
