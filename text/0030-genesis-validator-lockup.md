# Lock-up of genesis validators and access to tokens staked at genesis

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0030](https://github.com/casperlabs/ceps/pull/0030)

We propose that the operation of the auction and related contracts be modified to reflect the language of agreements regarding token allocations. To this end, we propose that the unlock of tokens be gradual and take place over the course of 90 days. Further, we introduce a new type of deploy, the genesis validator token withdrawal, which would enable genesis validators to withdraw eligible tokens without paying for gas.

## Motivation

[motivation]: #motivation

Tokens held by genesis validators are subject to legal agreements that specify an unlock period. Presently, the auction contract only implements a "cliff" at 90 days, but the unlock is expected to take place over the following 90 days, rather than instantly. Additionally, this proposal aims to resolve the expected shoftfall of circulating tokens at launch, as all amounts allocated to genesis validators will be staked. To this end, we propose introduction of a new time-limited deploy type specifically for reward withdrawals.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

We introduce two changes to the platform. The first change concerns the lock-in period and auto-win conditions managed by the auction contract:

- Total lock-in for 90 days remains in place
- After 90 days, tokens unlock gradually (equal proportion of starting amount) over the course of a further 90 days, possibly as weekly lump sums
- Auto-win conditions is set to false at the 90 day mark and genesis validators begin to compete in the auction

The second change concerns the problem of genesis validators having no tokens to withdraw their rewards. To this end, we propose introducing a new deploy type, the genesis validator token withdrawal, which requires no payment for gas and can be processed by the platform for the first **180 days**.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

N/a

## Drawbacks

[drawbacks]: #drawbacks

N/a

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

N/a

## Prior art

[prior-art]: #prior-art

No prior art is known.

## Future possibilities

[future-possibilities]: #future-possibilities

No further work is expected on this feature.
