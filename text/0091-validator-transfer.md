# Validator bid transfer

## Summary

[summary]: #summary

CEP PR: [casper-network/ceps#91](https://github.com/casper-network/ceps/pull/91)

We propose giving operator an ability to "transfer" their validator bid from one public key to another.

## Motivation

[motivation]: #motivation

Validator bid transfer could be used to securely transfer ownership of a validator bid without exchanging secret keys.
Another use case would be migrating to a different machine without any keys ever leaving the original host.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Our proposal is to add a new entry point to the auction contract - `change_bid_public_key`. 
This entrypoint would take public keys of a current validator bid owner and the transfer target. 
It would then create a `Bridge` record in place of the existing `ValidatorBid` pointing to a bid using the new public key. 
All relevant `Delegator` entries would be updated to use the new validator public key.
This would in effect transfer ownership of the current validator bid, it's stake and all delegators to the new address.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Chainspec update

Adding a new entry point to the auction contract would also require updating the chainspec to include the cost of calling the new method in the `system_costs.auction_costs` section.

### Adding `BidKind::Bridge` variant

The new `BidKind` enum variant would follow the same pattern as existing variants:

```rust
pub enum BidKind {
    ...
    /// A bridge record pointing to a new `ValidatorBid` after the public key was changed.
    Bridge(Box<Bridge>),
}
```

The `Bridge` struct would contain following data:

```rust
pub struct Bridge {
    /// Previous validator public key associated with the bid.
    old_validator_public_key: PublicKey,
    /// New validator public key associated with the bid.
    new_validator_public_key: PublicKey,
    /// Era when bridge record was created.
    era_id: EraId,
}
```

This would allow us to link previous `ValidatorBid` entries to current ones. 
Additionally storing the `era_id` allows us to prune old bridge records at some point in the future.

### Adding `change_bid_public_key` entry point

Within the new entry point we'd first need to validate the caller, since the method would only be called by the account associated with an existing validator bid. 

Then we'd verify that a `ValidatorBid` for the target public key does not exist yet. A new `Error` variant would have to be introduced to indicate a conflicting bid already exists.

Next we would read the current `ValidatorBid` with all relevant `Delegator`s from global state.
A `Bridge` record would be created and stored at the same `BidAddr` as the old `ValidatorBid`.
The existing `ValidatorBid` struct would be updated to use the new public key and stored in global state at the new address.
All delegator bids would also be updated with the new `validator_public_key` value and stored under updated addresses. 
Previous delegator bids would have to be pruned after processing. 

This process is somewhat similar to the way delegator bids are currently processed within the [`withdraw_bid`](https://github.com/teonite/casper-node/blob/6d028df56ca7db2edc714603344c8888cb9e0e0e/execution_engine/src/system/auction.rs#L171) entry point or in the [`ValidatorBid` migration](https://github.com/teonite/casper-node/blob/6d028df56ca7db2edc714603344c8888cb9e0e0e/execution_engine/src/engine_state/mod.rs#L504) during protocol upgrade.

### Handling `Bridge` records

Bridge records will have to be followed to determine the current public key associated with a validator in following workflows:

- distributing delegator rewards
- processing unbonds

To avoid potential unbounded computation issues (a looping bridge record chain) we propose limiting the max bridge record chain
length to 20. This should handle all reasonably expected use cases, since currently a bid would have to be changed 3 times in each era 
for an `UnbondingPurse` to become unprocessable.

Additionally, we propose setting the cost of calling the `change_bid_public_key` entrypoint to twice that of `add_bid` to disincentivize
using it as a potential attack vector.


## Drawbacks

[drawbacks]: #drawbacks

### No explicit delegator consent

Introducing this functionality would mean modifying delegators' bids without their explicit consent.
Until now this could only happen if a validator completely unstaked and delegations had to be returned.

### Unprocessable entities

While extremely unlikely, bridge record chains can potentialy introduce an uprocessable entity into the system. This must be handled to avoid
breaking node functionality - a bridge record chain length error should be handled gracefully and not break the rewards distribution process.

### Missing tokens

If an active validator creates an unprocessable chain of bridge records it would make it impossible to distribute rewards
and thus a lower than expected number of tokens would be minted. We believe that due to the resources needed the likelihood
of this impacting the economy is effectively zero.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Current solutions require moving private keys themselves to a different machine. While quicker and easier this approach is by definition unsafe, since it creates an opportunity for the keys to be stolen. 
In case of changing ownership there is also no guarantee that the original owner does not still have a copy.

Implementing the ownership transfer at protocol-level makes this process secure bacause the keys never have to leave the node.

## Prior art

[prior-art]: #prior-art

We are not yet aware of other solutions.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

There are no additional unresolved questions at the moment.
