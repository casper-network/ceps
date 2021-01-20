# Designated node operators

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0031](https://github.com/casperlabs/ceps/pull/0031)

We propose that validators be given the capability to designate a public key different from their own, corresponding to a private key of an operator designated to sign messages, thereby enabling validators to outsource their functions to third party operators.

## Motivation

[motivation]: #motivation

It is expected that many validators will want to rely on third parties to provision and operate nodes. 

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

At bid creation, the prospective validator may choose to specify an operator, by passing in the operator's public key. If the bid wins the auction, the operator will be responsible for creating all messages, signing with own private key. However, the validator will collect all of the rewards and would be subject to slashing, as usual. Our assumption is that an off-chain contractual relationship provides for the operator's compensation. Existing validators and bidders are able to switch operators at will, with the change taking effect in the first future era where the new operator is specified in a validator's winning bid. There is no confirmation for the relationship, but provisions have to be made to restrict designation to simplify reasoning about the behavior of the validator-operator pair. 

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### User workflow changes

Add an additional field, expecting a public key (or some type wrapping a public key), to the `add_bid` entry point of the auction contract. The supplied public key is expected to be part of a key pair belonging to a third party ("third party" can be the validator itself, if the field is not optional). The corresponding private key will be used to sign consensus messages on the validator's behalf, as well as sign for the validator in any other instances required by the protocol proper (i.e., besides user-initiated interactions with the auction and related system contracts). The link between the validator and the operator, as well as the history of validator's operator public keys, is to be tracked by the auction contract. Tracking of the pairings between a validator and its operators is intended to enable slashing, distribution of rewards and collection of transaction fees.

For safety, consensus messages from a validator signed with validator's own private key ought to be ignored in an era where this validator has a designated operator.

### Cases

Genesis validators - operator specified at platform launch

Entrant (no history as bidder or validator) - optionally supplies operator public key in `add_bid`

Existing bid - changes bid data (by using `add_bid`, similar to adjusting bid token amount) to include an operator public key; platform will expect this operator to sign messages on validator's behalf in a future era where this bid wins

### Typical timeline

Bid entered by a new entrant B, no operator provided

... (some blocks are added to the linear chain)

Auction for era N + `auction_delay` takes place, B wins
...

Era N + `auction_delay` starts, B is a validator, signing own consensus messages with own private key

During era N + `auction_delay`, B calls `add_bid` to supply an operator public key (and possibly add/reduce number of tokens in the bid)

Auction for era N + `auction_delay` + `auction_delay` takes place, B wins again

...

Era N + `auction_delay` + `auction_delay` starts, B is a validator, but consensus messages are produced and signed by the operator, consensus messages signed by B are ignored

### Restrictions on designation

In order to simplify reasoning about validator-operator pair behavior and to prevent potential introduction of equivocation or reward bugs, operator designation should be checked against the following:

1. Validator public keys slated to participate in future eras, other than that of the designator.
2. Validator public keys that are participating in current era or have participated in past eras, up to the unbonding wait period, other than that of the designator.
3. Bidder public keys for the upcoming auction, other than that of the designator.
3. Operator public keys corresponding to validator public keys in categories 1 through 3.

Should a designating deploy specify a "duplicate" public key contained in the categories 1 through 4, the deploy ought to be rejected. Note that we allow for the possibility of designating oneself as the operator, with no changes to current node or consensus behavior.

## Drawbacks

[drawbacks]: #drawbacks

We expect no downsides to introducing this UX feature.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

See Motivation.

## Prior art

[prior-art]: #prior-art

No prior art is known.

## Future possibilities

[future-possibilities]: #future-possibilities

We do not expect any further development of this feature at this time.
