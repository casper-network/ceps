# Configurable minimum/maximum delegation limits

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

We propose giving validators the ability to configure a minimum and maximum delegation amount which would then be enforced for their delegators.

## Motivation

[motivation]: #motivation

The chainspec already includes a global `minimum_delegation_amount` option, but adding validator-level limits would enable operators to more efficiently manage their delegators.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The auction contract already gives validators the ability to configure their bid by specifying a `delegation_rate` as an argument of the `add_bid` entry point.

Our proposal would follow a similar pattern and add two optional arguments to `add_bid`:

- `minimum_delegation_amount`
- `maximum_delegation_amount`

Since the arguments are optional, in case the user does not specify a value we'll use global limits as defaults. This requires adding a new chainspec option - `maximum_delegation_amount`.

Delegation limits would be used to validate new delegation requests in the `delegate` entry point of the auction contract.

As part of setting the limits we'd also forcibly unstake existing delegators whose bids are outside the specified range.

## TODO: Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Chainspec update

To make sure delegation limits are always available as part of `ValidatorBid` we need to add a `maximum_delegation_amount` chainspec option and update related structs (`EngineConfig` and `CoreConfig`).
This would set a global maximum as a conterpart to the already existing `minimum_delegation_amount`.

### Extending `ValidatorBid` struct

### Adding optional runtime args to `add_bid`

### Forced undelegation

### Additional validation in `delegate`

## Drawbacks

[drawbacks]: #drawbacks

This modification would force operators to keep track of their configuration settings when increasing/decreasing their stake and could potentialy lead to unintended misconfigurations.

If we implement forced unstaking of delegators it would introduce a situation when a delegators bid is modified without their knowledge. Until now this could only happen during validator unbonding.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

An alternative to modifying the `add_bid` entrypoint would be to introduce a dedicated entry point which would be used to set configuration options for existing bids.

This solution was discarded in part due to large functional overlap between both entry points 
but also because introducing more granular entry points would lead to inefficient global state modifications.

## TODO: Prior art

[prior-art]: #prior-art


## Unresolved questions

[unresolved-questions]: #unresolved-questions

### Forced undelegation

Do we want to forcibly unstake delegators? An alternative would be to leave existing delegators as-is and only validate new delegation requests.

### Staking rewards

A delegator could go over the maximum delegation limit after receiving their staking rewards. In this case we can transfer rewards over the limit to their main purse or allow them to go over the limit similarly to the point above.

## Future possibilities

[future-possibilities]: #future-possibilities

Future proposals to give validators more configuration options will probably follow a broadly similar pattern to this CEP.
