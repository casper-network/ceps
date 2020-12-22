# Disabling Bids on Liveness Failures

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/27)

This CEP outlines the course of action required for preventing liveness failures in the network when validators disconnect without retracting their bids.

## Motivation

[motivation]: #motivation

The Casper network only makes progress as long as enough validators (by weight) are correctly functioning. If too many of them stop participating in consensus, the weight of the remaining active ones may not be enough to finalize blocks anymore and the network will stall. In such a case, under the current design of the code, we have an irrecoverable failure: unless a manual intervention is involved, the network loses the ability to make any decisions at all, which means it can't fix itself, either.

In order to prevent such scenarios, we need to detect liveness failures in validators and disable their bids in the auction contract, so that inactive validators won't count towards the total weight and the percentage of active validators is as high as possible.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The first step towards disabling inactive validators is detecting them. In order to do that, we need to define what inactivity means.

We should note that the definition of inactivity might depend on the consensus protocol. In this CEP, we will focus on the case of the Highway protocol, which is the one currently used in the Casper network.

Within the Highway protocol, every validator assigns itself a round exponent, which it announces in its consensus messages. The round exponent defines the timeframes of rounds of the consensus protocol, and a validator is expected to send 2 or 3 messages every round (depending on whether it was the round leader or not), in accordance with its own round exponent.

With this in mind, we can define two notions as follows:

- a validator is **failing** if it sent less messages than expected within `k` out of the last `N` rounds.
- a validator is **inactive** if it sent _no_ messages within last `m` rounds.

Parameters `k`, `N` and `m` should be chosen in a way that achieves the best balance between false positives and false negatives. Since it is hard to predict in advance what such values could be, this CEP proposes making these configurable in the chainspec.

An inactive validator is probably one that completely lost the connection to the network. A failing validator either experiences intermittent network problems, or is deliberately trying to disrupt the network by sending messages erratically.

In either case, we would want to disable such a validator's bid. This would be done by a new type of system transaction, akin to rewards and slashing. Though, in order to minimize the time to action, such transactions would be included and executed in every block, instead of just in the switch block, like rewards and slashing.

However, the new type of transaction cannot work just as a regular transaction, in that it is not enough that the leader of the round proposes it in a block. If a single node considers another one inactive/failing, it doesn't mean that everyone or even that a majority of others will. Thus, such a transaction should be gossipped like a deploy, and it should be signed by nodes who agree that the offender should be disabled. Only if we collect enough of the signatures, the transaction should be included in a block (along with all the signatures proving that the validators agree about its correctness).

The new transaction would completely disable the validator's bid immediately upon execution and remove it from the validator set from the next era onwards. The funds would not be redeemable until the regular delay (`LOCKED_FUNDS_PERIOD`) passes. If the validator desires to become a validator again, it would have to submit a new, separate bid.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The Highway implementation would keep track of which validators perform as expected, that is, which of them are consistently sending the correct number of messages.

The struct `Validator` in `src/components/consensus/highway_core/validators.rs` would contain an additional field:

```rust
struct Validator<VID> {
    // ...
    state: ValidatorState,
}
```

`ValidatorState` would be the following enum:

```rust
enum ValidatorState {
    /// A state meaning that the validator is active so far, but tracking rounds that didn't meet the expectations
    Active {
        inactive_rounds: u8, // or a larger type, if `m` could be larger than 255
        failing_rounds: VecDeque<bool>,
    },
    /// A state meaning that the validator is now considered failing
    Failing,
    /// A state meaning that the validator is now considered inactive
    Inactive,
}
```

As the rounds pass, the instance would update the state for every validator participating in the protocol. If a validator becomes failing or inactive, the Highway protocol would emit a new variant of `ProtocolOutcome`:

```rust
enum ProtocolOutcome<I, C: Context> {
    // ...
    ValidatorInactive(C::ValidatorId),
}
```

The Era Supervisor, upon receiving such an outcome, would generate a new variant of `ConsensusMessage` and request gossipping of this message:

```rust
enum ConsensusMessage {
    // ...
    ValidatorInactive {
        disabling_transaction: Deploy,
    }
}
```

The `disabling_transaction` would consist of:

- `payment`: empty `ExecutableDeployItem::ModuleBytes`
- `session`: a new variant `ExecutableDeployItem::DisableBid { era_id: EraId, validator: PublicKey }` with `era_id` being set to the ID of the era that emitted the `ValidatorInactive` outcome
- `approvals`: a vector containing a single approval by the validator that generated the deploy.

Other validators, when receiving `ConsensusMessage::ValidatorInactive`, would collect the valid approvals in a new field in `EraSupervisor`:

```rust
struct EraSupervisor<I> {
    // ...
    collecting_approvals: BTreeMap<(EraId, PublicKey), Deploy>,
}
```

where the key would be the same data as in `ExecutableDeployItem::DisableBid`, and the value would be the full deploy, with all the approvals collected so far.

Every validator, when it is their turn to propose a block, would check `collecting_approvals` for deploys that got enough approvals. Any such deploys would be included in the proposed block.

`ExecutableDeployItem::DisableBid`, upon execution, would cause a call into the auction contract, which would remove the relevant validator's bid from eras `era_id + 1` onwards. The funds would remain locked for `LOCKED_FUNDS_PERIOD`.

## Drawbacks

[drawbacks]: #drawbacks

If we are too eager with disabling bids, we could inadvertently punish honest validators just having network issues. This can be mitigated by adjusting the choice of the relevant constants, though.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Instead of gossipping signatures for a new deploy, disabling bids could be done based on the past cone of a block in the Highway DAG. It could be inferred from the DAG which validators were inactive and for how long. However, it is unclear whether malicious validators couldn't collaborate in order to deliberately create outdated panoramas and make it look as if another validator was inactive. We shouldn't use this approach unless we can prove it is safe.

There is also another potential approach in regards to the precise mechanism of disabling bids. Instead of being removed entirely, the bid could be flagged as inactive, and the flag could be removed when the validator comes back online. However, this would open the network up to being exploited by "lazy validators", where a validator could go offline for large amounts of time, get disabled, then reenabled immediately when it comes back online.

## Prior art

[prior-art]: #prior-art

It is a general practice in networked applications to disconnect unresponsive nodes.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- Should inactive validators be treated differently from failing ones?
- Should we even disable the bids of failing validators?
- Should we slash parts of the bids of the validators we disable?
- Can we remove the validator from the set of validators for the next era, or do we have to observe the `AUCTION_DELAY` here?

## Future possibilities

[future-possibilities]: #future-possibilities

None at the moment.
