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
It would then modify the existing `ValidatorBid` and all related `Delegator` entries to use the new validator public key.
This would in effect transfer ownership of the current validator bid, it's stake and all delegators to the new address.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Chainspec update

Adding a new entry point to the auction contract would also require updating the chainspec to include the cost of calling the new method in the `system_costs.auction_costs` section.

### Adding `change_bid_public_key` entry point

Within the new entry point we'd first need to validate the caller, since the method would only be called by the account associated with an existing validator bid. 

Then we'd verify that a `ValidatorBid` for the target public key does not exist yet. A new `Error` variant would have to be introduced to indicate a conflicting bid already exists.

Next we would read the current `ValidatorBid` with all relevant `Delegator`s from global state, update them with the new `validator_public_key` value and store them. Previous delegator bids would have to be pruned after processing. 

This process is somewhat similar to the way delegator bids are currently processed within the [`withdraw_bid`](https://github.com/teonite/casper-node/blob/6d028df56ca7db2edc714603344c8888cb9e0e0e/execution_engine/src/system/auction.rs#L171) entry point or in the [`ValidatorBid` migration](https://github.com/teonite/casper-node/blob/6d028df56ca7db2edc714603344c8888cb9e0e0e/execution_engine/src/engine_state/mod.rs#L504) during protocol upgrade.


## Drawbacks

[drawbacks]: #drawbacks

Introducing this functionality would mean modifying delegators' bids without their explicit consent.
Until now this could only happen if a validator completely unstaked and delegations had to be returned.

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
