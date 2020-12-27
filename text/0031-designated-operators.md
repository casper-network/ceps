# Designated node operators

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0031](https://github.com/casperlabs/ceps/pull/0031)

We propose that validators be given the capability to designate a public key different from their own to sign messages, thereby enabling validators to outsource their functions to third party operators.

## Motivation

[motivation]: #motivation

It is expected that many validators will want to rely on third parties to provision and operate nodes. 

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Add an additional field, expecting a public key (or some type wrapping a public key), to the `add_bid` entry point of the auction contract. The supplied public key will be used to sign consensus messages on the validator's behalf and to sign for or identify the validator in all instances required by the protocol proper. The link between the validator and the operator, as well as the history of validator's operator public keys, is to be tracked by the auction contract. Tracking of the pairings between a validator and its operators is intended to enable slashing, distribution of rewards and collection of transaction fees.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

N/a

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