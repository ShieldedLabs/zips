    ZIP: Unassigned
    Title: Crosslink Proof-of-Stake and Validator Staking
    Owners: Sam H. Smith <sam@shieldedlabs.net>
            Phillip Lujan <phillip@shieldedlabs.net>
            Mark Henderson <mark@shieldedlabs.net>
    Credits: Zooko Wilcox
             Andrew Mayerczyk
    Status: Draft
    Category: Consensus
    Created: 2026-04-09
    License: MIT
    Pull-Request: <https://github.com/zcash/zips/pull/???>


# Terminology

The key words "MUST", "REQUIRED", "MUST NOT", "SHOULD", AND "MAY" in this
document are to be interpreted as described in BCP 14 [^BCP14] when, and
only when, they appear in all capitals.

The character &sect; is used when referring to sections of the Zcash Protocol Specification. [^protocol]

The terms below are to be interpreted as follows:

Validator (Finalizer)

: A participant in the Crosslink BFT consensus protocol, identified by an Ed25519 public key and weighted by staked ZEC.

Delegator

: A ZEC holder who delegates stake to a validator via a delegation bond, sharing in rewards without running consensus infrastructure.

Delegation Bond

: A time-locked stake commitment that grants voting power to a validator. Bonds have an unbonding period before funds can be withdrawn.

Roster

: The set of all validators with non-zero stake. The "active roster" is the subset of at most 100 validators that participate in consensus at a given BFT height, selected by descending stake.

Staking Action

: A transaction field that encodes a staking operation (bond creation, unbonding, withdrawal, validator registration, etc.).

Unbonding Period

: The mandatory delay between initiating bond withdrawal and receiving funds. During unbonding, the stake does not participate in consensus but remains subject to slashing.


# Abstract

This ZIP specifies the Proof-of-Stake staking protocol for Zcash Crosslink. It defines the on-chain transaction format for staking operations, the validator lifecycle (registration, delegation, unbonding, withdrawal), and the roster construction algorithm that determines which validators participate in BFT consensus.

The staking protocol provides the economic security layer for Crosslink: validators stake ZEC as collateral, earn rewards for honest participation, and risk slashing for equivocation.


# Motivation

The Crosslink BFT finality layer requires a set of validators weighted by economic stake. This stake serves two purposes:

1. **Sybil resistance.** Stake-weighting prevents an attacker from cheaply creating many validator identities to control the consensus.
2. **Economic security.** Staked ZEC can be slashed for provable misbehavior (equivocation), creating a concrete economic cost for attacks on finality.

The staking protocol must integrate cleanly with Zcash's existing transaction model while supporting the full validator lifecycle: registration, delegation, reward distribution, key rotation, and graceful exit.


# Privacy Implications

Staking operations are inherently transparent:

1. Validator public keys are published on-chain and in finality certificates.
2. Delegation bonds are visible in transactions.
3. Stake amounts determine voting power and are publicly verifiable.

This transparency is necessary for the security model &mdash; anyone must be able to verify that a finality certificate represents more than 2/3 of stake. However, it means that validators and large delegators sacrifice financial privacy for the staking portion of their ZEC.

Delegators who wish to maintain privacy for non-staked funds SHOULD use separate addresses for staking and shielded transactions.


# Requirements

1. On-chain staking operations encoded in standard Zcash transactions.
2. Validator registration with Ed25519 key binding.
3. Delegation of stake from delegators to validators.
4. Unbonding period for withdrawals.
5. Roster construction capped at 100 active validators.
6. Key rotation for operational security.


# Non-requirements

1. **Reward distribution formula.** The precise reward schedule (inflation, fee sharing, etc.) is deferred to a separate ZIP or the network upgrade specification.
2. **Slashing conditions and penalties.** While the staking protocol provides the mechanism for slashing (reducing a validator's bonded stake), the specific conditions (equivocation evidence, liveness faults) and penalty amounts are specified separately.
3. **Governance.** Staked ZEC does not confer governance rights beyond consensus participation.


# Specification

## Staking Actions

Staking operations are encoded as an optional `StakingAction` field in Zcash transactions. A transaction MAY contain at most one staking action.

### StakingAction Structure

| Field | Type | Size | Description |
|-------|------|------|-------------|
| kind | u8 | 1 | Staking action type (see below) |
| pub_key | [u8; 32] | 32 | Ed25519 public key of the target validator |
| amount | u64 LE | 8 | Amount in zatoshis (where applicable) |
| payload | bytes | variable | Action-specific data |

### Staking Action Types

| Kind | Value | Name | Description |
|------|-------|------|-------------|
| Null | 0 | &mdash; | No staking action (default for non-staking transactions) |
| CreateNewDelegationBond | 1 | Bond | Create a new delegation bond to a validator |
| BeginDelegationUnbonding | 2 | Unbond | Begin the unbonding period for a delegation bond |
| WithdrawDelegationBond | 3 | Withdraw | Withdraw funds after unbonding period completes |
| RetargetDelegationBond | 4 | Retarget | Move delegation from one validator to another |
| RegisterFinalizer | 5 | Register | Register a new validator with an Ed25519 key |
| ConvertFinalizerRewardToDelegationBond | 6 | Reinvest | Convert accumulated validator rewards to a new bond |
| UpdateFinalizerKey | 7 | Rotate | Update a validator's Ed25519 signing key |

### Action Semantics

#### CreateNewDelegationBond (kind = 1)

Creates a new delegation bond, locking `amount` zatoshis and granting voting power to the validator identified by `pub_key`.

**Preconditions:**
- The validator identified by `pub_key` MUST be registered.
- The transaction MUST contain transparent or shielded inputs totaling at least `amount`.
- `amount` MUST be greater than zero.

**Effects:**
- Creates a new bond record associating the delegator's transaction with the validator.
- The validator's `voting_power` increases by `amount`.
- The funds are locked and cannot be spent until unbonding completes.

#### BeginDelegationUnbonding (kind = 2)

Initiates the unbonding process for an existing delegation bond.

**Preconditions:**
- A bond from the caller to the specified validator MUST exist.
- The bond MUST NOT already be in the unbonding state.

**Effects:**
- The bond enters the "unbonding" state.
- The validator's `voting_power` decreases by the bond amount.
- The unbonding period timer begins. During this period, the stake is still subject to slashing.

#### WithdrawDelegationBond (kind = 3)

Withdraws funds from a fully unbonded delegation bond.

**Preconditions:**
- The bond MUST be in the "unbonded" state (unbonding period complete).

**Effects:**
- The bond is deleted.
- The locked funds are released to the withdrawer.

#### RetargetDelegationBond (kind = 4)

Moves an existing delegation bond from one validator to another without unbonding.

**Preconditions:**
- A bond to the source validator MUST exist.
- The target validator identified by `pub_key` MUST be registered.

**Effects:**
- The source validator's `voting_power` decreases by the bond amount.
- The target validator's `voting_power` increases by the bond amount.
- The bond record is updated to reference the target validator.

#### RegisterFinalizer (kind = 5)

Registers a new validator with the given Ed25519 public key.

**Preconditions:**
- No validator with the same `pub_key` MUST be currently registered.
- The transaction SHOULD include an initial self-delegation bond (though this is not strictly required).

**Effects:**
- A new validator entry is created with `voting_power = 0`.
- The validator becomes eligible to receive delegation bonds.

#### ConvertFinalizerRewardToDelegationBond (kind = 6)

Converts accumulated consensus rewards for a validator into a new delegation bond, compounding stake.

**Preconditions:**
- The validator MUST have accumulated rewards.

**Effects:**
- Accumulated rewards are converted to a delegation bond to the same validator.
- `voting_power` increases by the reward amount.

#### UpdateFinalizerKey (kind = 7)

Rotates a validator's Ed25519 signing key.

**Preconditions:**
- The transaction MUST be authorized by the validator's current key (e.g., signed by the old key, or the transaction spends a UTXO controlled by the validator's registered address).
- The new `pub_key` MUST NOT already be registered.

**Effects:**
- The validator's signing key is updated to the new `pub_key`.
- All existing delegation bonds are preserved; they now delegate to the new key.
- The old key is no longer valid for BFT consensus participation.

## Roster Member

Each validator in the roster is represented as:

| Field | Type | Size | Description |
|-------|------|------|-------------|
| pub_key | [u8; 32] | 32 | Ed25519 public key |
| voting_power | u64 LE | 8 | Total delegated stake in zatoshis |
| txids | StakeTxId[] | variable | Transaction IDs of delegation bonds |

### Serialization

```
pub_key:       32 bytes
voting_power:  8 bytes (u64 LE)
txid_count:    8 bytes (u64 LE)
txids:         txid_count * sizeof(StakeTxId)
```

## Roster Construction

The active roster for a given BFT height is constructed as follows:

1. Collect all registered validators with `voting_power > 0`.
2. Sort by public key (lexicographic, ascending).
3. Truncate to at most 100 validators.
4. Compute `cumulative_stake` for each validator: the running sum of `voting_power` for validators 0 through i.

The resulting `SortedRosterMember` list is used by Tenderlink for:

- Proposer selection (BLAKE3 hash mod total_stake, binary search on cumulative_stake).
- Threshold computation (2f+1 for decisions, f+1 for round catch-up).
- Finality certificate validation (checking that signers are roster members with sufficient total stake).

### Roster Updates

The roster is updated after each BFT height decision:

1. The decided BFT block is applied to the PoW state.
2. Any staking actions in PoW blocks up to the new finalized height are processed.
3. The updated roster is passed to Tenderlink for the next height.

This means roster changes take effect with a delay of at least &sigma; PoW blocks (the confirmation depth), providing stability during consensus rounds.

## Stake Tracking

Delegation bonds are tracked by their originating transaction ID (`StakeTxId`). This provides:

1. **Auditability.** Anyone can trace a validator's stake to specific on-chain transactions.
2. **Granular unbonding.** Individual bonds can be unbonded independently.
3. **Slashing precision.** Specific bonds can be slashed rather than the validator's entire stake.


# Rationale

**On-chain staking actions:** Encoding staking operations as transaction fields (rather than separate transaction types) allows staking to compose with existing Zcash transaction features (transparent inputs, memo fields, etc.) without requiring a new transaction version.

**Delegation model:** Separating validators (who run consensus infrastructure) from delegators (who provide economic security) allows broad participation in staking without requiring every ZEC holder to operate a validator node. This is the standard approach in modern PoS protocols (Cosmos, Ethereum 2.0, etc.).

**100-validator cap:** Limiting the active roster to 100 validators bounds the size of finality certificates and the cost of consensus. With 100 validators, each producing a 96-byte signature, a finality certificate is approximately 9.6 KB &mdash; large but manageable.

**Unbonding period:** The unbonding period prevents "stake grinding" attacks where a validator misbehaves and immediately withdraws stake before slashing evidence is processed. The specific duration is a parameter to be determined based on security analysis.

**Key rotation:** Validator key rotation is essential for operational security. Keys may be compromised, hardware may be replaced, or validators may wish to migrate to new infrastructure. The `UpdateFinalizerKey` action supports this without disrupting delegation relationships.

**StakeTxId tracking:** Tracking individual bond transactions (rather than just aggregate voting power) enables fine-grained unbonding and slashing. A delegator can unbond a specific bond without affecting their other delegations to the same validator.


# Deployment

The staking protocol is activated as part of the Crosslink network upgrade. Staking transactions MUST be accepted in PoW blocks starting from a pre-activation registration period (before Crosslink activation) to allow the initial validator set to form.

The registration period SHOULD begin at least [TBD] blocks before the Crosslink activation height, giving validators time to register and accumulate stake.


# Reference implementation

- [crosslink_monolith/librustzcash/zcash_primitives/src/transaction/mod.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; StakingAction, StakingActionKind, RosterMember
- [crosslink_monolith/zebra-crosslink/zebra-crosslink/src/lib.rs](https://github.com/AzureMarker/crosslink_monolith) &mdash; TFLServiceInternal roster management


# Open issues

1. **Reward distribution.** The mechanism for computing and distributing consensus rewards (block rewards, transaction fees, inflation) to validators and delegators is not yet specified.
2. **Slashing protocol.** The conditions for slashing (equivocation evidence, liveness faults), the penalty amounts, and the mechanism for submitting and verifying slashing evidence need formal specification.
3. **Unbonding duration.** The length of the unbonding period is not yet determined. It must be long enough to detect and process misbehavior but short enough to not discourage participation.
4. **Minimum stake.** Whether there is a minimum stake for validators and/or delegation bonds is not yet specified.
5. **Validator cap selection.** The 100-validator cap is a prototype value. The production value should be determined by analysis of the trade-off between decentralization (more validators) and efficiency (smaller finality certificates, faster consensus).
6. **Transparent-only staking.** The current implementation appears to use transparent (not shielded) staking transactions. Whether shielded staking is feasible or desirable requires analysis of the privacy and verifiability trade-offs.
7. **Commission model.** Validators may wish to charge delegators a commission on rewards. The commission structure is not yet specified.
8. **Jailing.** Validators that are offline for extended periods may need to be "jailed" (temporarily removed from the active roster) to maintain liveness. The jailing mechanism is not yet specified.
9. **Re-delegation without unbonding.** The `RetargetDelegationBond` action allows instant re-delegation. Whether this should require an unbonding period (to prevent gaming) needs analysis.
10. **Governance integration.** Whether staked ZEC carries governance weight (for ZCG, protocol parameter changes, etc.) is a policy question beyond this ZIP's scope.


# References

[^BCP14]: [Information on BCP 14 &mdash; "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^tfl-book]: [Zcash Trailing Finality Layer, Electric Coin Company](https://electric-coin-company.github.io/tfl-book/)

[^zip-tenderlink]: [ZIP-XXX: Tenderlink: BFT Consensus Protocol for Crosslink](draft-crosslink-tenderlink)

[^zip-crosslink-architecture]: [ZIP-XXX: Crosslink Network Architecture](draft-crosslink-architecture)

[^zip-crosslink-transport]: [ZIP-XXX: Crosslink Transport Protocol (STP)](draft-crosslink-transport)
