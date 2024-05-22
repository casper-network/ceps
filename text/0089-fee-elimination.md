# Elimination of gas fees

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0089](https://github.com/casperlabs/ceps/pull/89)

We propose introduction of a delayed gas fee refund feature, replacing the (currently inactive) refund feature.

## Motivation

[motivation]: #motivation

Due to the inherent uncertainty in gas costs caused by variation in inputs, users almost always need to overclaim gas to eliminate risk of reversion. Because as-implemented instant refunds are currently set to 0, this creates a serious UX problem, as users must almost always overpay to guarantee successful execution. Unfortunately, given the execution-after-consensus paradigm Casper uses, meaningful non-zero refunds create a serious attack vector, enabling malicious or careless users to fill up blocks with overclaimed gas.

To address both the UX issue and the security issue with refunds, we propose a re-implementation of the refund feature, introducing delayed refunds with a percentage of "used" token made available to the user on a linear schedule. Unlike the current refund feature, the new system will functionally refund all token and not just token paid for overclaimed gas.

Note that we use the term "refund" for simplicity and to emphasize the viewpoint of the user. However, token "spent" under the new system will not be transferred or otherwise associated with any account other than that of the user.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

### System view

With the implementation of this proposal, execution of a deploy will still trigger the code pathways that calculate expended gas with a given gas-token conversion rate to produce a "fee." However, no transfer to the validator will take place, leaving the token that would have been expended by the user still associated with their account, but under a temporary hold. The "used" token is released for further use on a linear schedule as a function of block time.

### User view

From the user's perspective, all token that would, under the current system, be paid to the validator are eventually "refunded." Token are made available on a linear schedule as a function of block time.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Account level lazy tracking

The logical account object records the "spend" on execution from deploys signed by the account's private key on a per-purse
basis. Whenever token from some purse is to be spent in any manner, a balance check is performed, taking into account 

- unencumbered purse token
- the individual purse's holds
- block time (which is known to the execution engine)

The available balance consists of 

- unencumbered purse token 
- token from expired holds
- token from continuing holds subject to a linear release schedule parametrized in the chainspec

Optimization is left as an implementation detail, e.g., due to the inexorable march of time, all the holds' expiration times are naturally pre-sorted, enabling a quick disposal of old holds in the balance check calculation.

### Changes to gas fee logic

No changes, at least due to the introduction of this feature, are expected to be made to the calculation of gas fees.

## Drawbacks

[drawbacks]: #drawbacks

Elimination of user fees paid to validators creates adverse incentives for validators. In the interest of keeping
this CEP concise, we are planning to introduce a companion incentive-aligning feature in a later CEP.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The basic requirement to charge for computation arises from the need to ration use of scarce resources, a matter made
more pressing by ever-shortening block times and public demand for low latency. No matter how the "fees" are structured,
eventually the cost must be a material one. With this feature, we replace immediate hard economic cost of losing the token permanently with a soft economic cost of losing the time value of token. At a minimum, this amounts to the foregone yields from staking the token instead of using it to "pay" for computation.

## Prior art

[prior-art]: #prior-art

We do not currently know of an example of another blockchain replacing the use of token as transferable payment for computation with the use of token as permanent claim to some portion of the blockchain's compute. 

## Future possibilities

[future-possibilities]: #future-possibilities

We expected to introduce further features to re-align validator incentives with gas fee elimination at a later date.