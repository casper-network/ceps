# Proportional cap on delegation

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0029](https://github.com/casperlabs/ceps/pull/0029)

We propose introduction of a delegation cap, or a proportional limit on delegation relative to own staked tokens.

## Motivation

[motivation]: #motivation

We expect all users of the platform to assess relevant risks of interacting with various features that we supply, as both the performance of the platform and the outcomes of any interaction are uncertain in a multi-user, concurrent environment. Consequently, we would normally expect that users avoid delegating to validators with a highly skewed ratio of delegated to own tokens, because such validators would sustain only weak penalties for equivocation relative to their weight. However, in order to enhance platform security, we propose to limit overdelegation using a soft cap that does not compromise platform UX and favors delegators over validators.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

We propose to limit the amount of delegated tokens contributing to a bid to a multiplier X of validator’s own token. For example, if a prospective validator’s standing bid is 100 tokens and X is set to 1.2, up to 20 tokens may be delegated. Should a bid be decreased to the point where the quantity of existing delegated tokens violates this constraint, delegated tokens above the limits, in aggregate, should no longer “count” for auction result determination or reward weight. 

However, for simplicity and encouragement of staking own tokens, distribution of rewards to delegators should remain proportional to their delegated tokens at all times. E.g., if a validator A bids 100 (setting his take from delegator rewards to 0) and receives delegations of 15 and 5 tokens from B and C, respectively, then withdraws 20 tokens, the ratio of validator to delegator rewards becomes 80/20 (before the bid decrease, it would have been 100/20), with B receiving 3 parts to each 1 part of C’s reward. However, the “weight” of the validator in auctions and reward calculations would now be 96 instead of 120. 

If there are delegated tokens above the allowable limit, further delegation is still permitted to prevent "races" between deploys and user frustration.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Chainspec parameters

#### Delegation cap multiplier

As described above, this parameter, set to a value always weakly greater than 1, would determine the allowable ratio of total tokens to validator's tokens in a bid. Assuming *x >= 1* is our parameter, *y* is the quantity delegated tokens and *z* is the quantity of validator's tokens, the relationship between the three variables is described as follows,

*z + y <= z \* x*

### Changes to auction and related system contracts

If the validator adjusts his bid in a way that causes the bid to violate the limit, then the weight of the bid (for purposes of protocol leader selection and auction operation) is capped by the multiplier applied to own tokens. Any rewards, however, are to be distributed between the validator and the delegator in accordance to the true quantities of tokens staked.

## Drawbacks

[drawbacks]: #drawbacks

The expected drawback to this proposal is the exclusion of high delegation ratio validators, which may make it harder for less risk-averse delegators to achieve desired risk exposure. For instance, a highly performant, unknown and token-poor validating node could be excluded from the market, thereby denying potential delegators outsized returns from delegating to such a node.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The rationale for this feature is our own risk-aversion. While we hope that all users act to further their rational, long-term self-interest, years of experience with Internet platforms suggest the propriety of certain safeguards that prevent users from making almost certain mistakes. It is also judged that there is a low probability but high magnitude risk of a PR event associated with a massively overdelegated validator triggering a slashing. Finally, the logic of consensus security intuitively demands that we require participants to have "skin in the game." 

We have considered rejecting delegations that would make a bid be subject to the soft cap, but decided against it, because of aforementioned deploy "races."

## Prior art

[prior-art]: #prior-art

No prior art is known.

## Future possibilities

[future-possibilities]: #future-possibilities

We do not expect any further development of this feature at this time.
