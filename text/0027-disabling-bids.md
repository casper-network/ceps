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

In either case, we would want to disable such a validator's bid. This would be done by a new type of system transaction, akin to rewards and slashing. Just like rewards and slashing, it would be included in the switch block and passed to the auction contract to be processed.

The validity of this transaction would be determined by the consensus state as seen by the switch block proposal consensus unit (i.e. the number of messages sent by each validator would be calculated based on what the unit containing the switch block sees) - this ensures that all validators will agree about whose bids need to be disabled.

The new transaction would completely disable the validator's bid upon execution by the auction contract, which would set a flag on the bid, marking it as inactive. This would cause the bid not to be taken into account in auctions for the eras starting at `AUCTION_DELAY` eras in the future. The bid would remain in this state until the validator either bids again (possibly a zero amount, which would remove the "inactivity" flag and make it eligible for winning future auctions again) or explicitly retracts the bid.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

A new field would be added to `consensus::consensus_protocol::EraEnd`:

```rust
struct EraEnd<VID> {
    // ...
    inactive_validators: Vec<VID>,
}
```

This field would be populated by the finality detector, just like the `rewards` field, based on the configured threshold for inactivity.

An analogous field would be added in the `StepRequest` structure, which is now used to signal the rewards and slashings to the auction contract. `BlockExecutor::execute_next_deploy_or_create_block` populate it based on `EraEnd::inactive_validators`.

The auction contract would then use this data to disable the bids as applicable.

## Drawbacks

[drawbacks]: #drawbacks

If we are too eager with disabling bids, we could inadvertently punish honest validators just having network issues. This can be mitigated by adjusting the choice of the relevant constants, though.

One important drawback of this approach is that the bids are only disabled once per era and observe the `AUCTION_DELAY`. This means that even if validators don't go offline at once, but enough of them become inactive during the span of `AUCTION_DELAY` eras, we will be left with no recourse but to wait until some of them come back online. It would be tempting to remove the validator from the validator set starting at the next era in order to quicken the process, but that has security implications - it would modify the set of validators in already decided eras, so there would be a possibility of an attacker having some control over the sequence of the round leaders if we did that.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Since disabling the bids only after `AUCTION_DELAY` eras still carries a risk of stalling the network in case of multiple validators becoming inactive, it would be desirable to evict inactive validators more quickly, eg. after one round. Unfortunately, there is no clear way of achieving this effect without making modifications having deeply reaching consequences, so this remains a future possibility for now.

There is also another potential approach in regards to the precise mechanism of disabling bids. Instead of being removed entirely, the bid could be flagged as inactive, and the flag could be removed when the validator comes back online. However, this would open the network up to being exploited by "lazy validators", where a validator could go offline for large amounts of time, get disabled, then reenabled immediately when it comes back online.

## Prior art

[prior-art]: #prior-art

It is a general practice in networked applications to disconnect unresponsive nodes.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- Should inactive validators be treated differently from failing ones?
- Should we even disable the bids of failing validators?
- Should we slash parts of the bids of the validators we disable?

## Future possibilities

[future-possibilities]: #future-possibilities

It might be possible to evict inactive validators more often than once per era. Possible approaches include:

- Ending eras early. When a validator becomes inactive, we end the era, include the inactivity information in the switch block and start a new one without the inactive validator.
- Ignoring the weight of inactive validators at the consensus level - not including them in the finality calculations, for example.

These approaches have far reaching consequences, though. Ending eras early has implications for security: it effectively modifies the set of validators for an era with a known seed, which means that an attacker could use this to manipulate the sequence of round leaders. The second approach, on the other hand, could have implications for the correctness of the consensus algorithm, so it would have to be formulated more precisely and investigated in much more depth.
