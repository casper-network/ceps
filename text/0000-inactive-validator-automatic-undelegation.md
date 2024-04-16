# Automatic undelegation from inactive nodes

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

We propose introducing a functionality of automatically undelegating all delegators from validators 
who have been inactive for a set number of eras.

Additionally, we propose giving validators an option to impose a more restrictive era limit on themselves,
as a guarantee for their delegators.

## Motivation

[motivation]: #motivation

Automatic undelegation is meant to encourage good stewardship of node operators. It would also make it easier 
and quicker for delegators to unlock their funds if staked with an inactive validator.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Our proposal is to extend the `ValidatorBid` struct to track the era when a given validator has become inactive.
We would then run a cleanup process at the end ef each era before the auction is run to find all validators
who have been inactive longer than the configured delay and undelegate all their delegators.

The undelegation would follow the existing approach - an `UnbondingPurse` would be created for each delegator's stake
and it would then be returned to the delegator's main purse after the unbonding delay.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Chainspec update

Chainspec needs to be updated to include the default global setting for `inactive_validator_undelegation_delay`.
Similar to `auction_delay` this setting would express a delay in eras after a validator becomes inactive before 
all delegators are undelegated.

### Extending `ValidatorBid` struct to include `eviction_era` and `inactive_validator_undelegation_delay`

To enable tracking how many eras have passed since a validator has been marked as inactive, the `ValidatorBid` struct 
needs to be extended to include an `eviction_era` field which would contain an optional `EraId` to track when a given
validator was evicted.

We would also add an `inactive_validator_undelegation_delay` field to enable configuration of validator-specific 
undelegation delay.

The changed struct would look as follows:

```rust
pub struct ValidatorBid {
    /// Validator public key
    validator_public_key: PublicKey,
    /// The purse that was used for bonding.
    bonding_purse: URef,
    /// The amount of tokens staked by a validator (not including delegators).
    staked_amount: U512,
    /// Delegation rate
    delegation_rate: DelegationRate,
    /// Vesting schedule for a genesis validator. `None` if non-genesis validator.
    vesting_schedule: Option<VestingSchedule>,
    /// `true` if validator has been "evicted"
    inactive: bool,
    /// Era when validator has been "evicted"
    eviction_era: Option<EraId>,
    /// Delay after validator has been "evicted"
    /// before all delegators are undelegated
    inactive_validator_undelegation_delay: u64,
}
```

### Adding optional argument to `add_bid` auction entrypoint

To give operators an option to configure a more restrictive delay setting on themselves we would add an optional
`u64` argument `inactive_validator_undelegation_delay` to the `add_bid` entrypoint of the Auction contract. If a value
is not provided the chainspec default would be used for a given validator bid. 

The value would also be validated to be smaller or equal to the global setting and throw a corresponding error if appropriate.

## Drawbacks

[drawbacks]: #drawbacks

Introducing this functionality would mean modifying delegators' bids without their explicit consent. 
Until now this could only happen if a validator completely unstaked and delegations had to be returned.

This could potentially introduce some confusion, since there's currently no way for a delegator to clearly identify
why a given transfer to their purse has happened. In the future this could be solved by leveraging the event system 
to emit events which could be correlated with a given transfer for example in a block explorer.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Since the overall functionality is mostly straightforward, there were no alternative approaches considered.

## Prior art

[prior-art]: #prior-art

We are not yet aware of other solutions.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

There are no outstanding questions at the moment.

## Future possibilities

[future-possibilities]: #future-possibilities

As mentioned above in the future we could add traceability to the system by using an event system.
If also implemented on the block explorer side this would create a better user experience for delegators.