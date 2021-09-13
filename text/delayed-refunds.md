# Delayed refunds

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps# ](https://github.com/casperlabs/ceps/pull/)

We propose enabling the dormant gas refund feature, additionally introducing a variable delay to disbursement of refunded tokens which depends on recently observed user behavior.

## Motivation

[motivation]: #motivation

Due to the inherent uncertainty in gas costs caused by variation in inputs, users almost always need to overclaim gas to eliminate risk of reversion. Because refunds are always set to 0, this creates a serious UX problem. Additionally, a complete resolution to this problem would require full refunds. Unfortunately, given the execution-after-consensus paradigm Casper uses, full refunds create a serious attack vector, enabling malicious or careless users to fill up blocks with overclaimed gas.

To address both the UX issue, and the security issue with refunds, we propose a delay to disbursement of refunded tokens, which would always be held for at least one era. However, depending on the proportion of gas refunded to the user, the delay would grow longer. The length of the delay is governed by the amount of gas recently refunded.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

### System view

With the implementation of this proposal, system contracts would maintain some metadata regarding accounts reflecting their recent usage of gas and penalizing poor gas estimation (high refunds) by delaying the disbursement of tokens due to be refunded. It is expected that the parameters would be set in such a way as to prevent this ever being an issue for a user with gas use estimates off by less than 10%, with such users receiving their entire refund at the end of each era.

### User view

From the user's perspective, the refunds are full but possibly delayed. It is expected the delayed refunds will incentivize use of whichever gas cost estimation tools may be available. Additionally, delayed refunds should disincentivize exploitation of full refunds for malicious purposes, since any attempt to maliciously overclaim gas at low cost would result in prohibitively lengthy refund disbursement schedules.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Refund delay logic

We will denote state variables evolving as eras switch in bold letters.

#### Elements

*Thresholds* in {p<sub>1</sub>, ..., p<sub>N</sub>} are percentages (fractions ranging from 0 to 1) that define ranges of observed past refunds, which determine a *penalty* in {1, ..., M}.

Fix N to be #{1, ..., M}. The thresholds define the penalty in the following way:

- Take p<sub>i</sub> and p<sub>i+1</sub>
- If one minus *aggregate utilization* **P** (1 - **P**) is between p<sub>i</sub> and p<sub>i+1</sub>

    - Penalty i applies
- If 1 - **P** is above p<sub>n</sub>, penalty M applies

We will use function *f* to represent the above mapping.

Let t denote the index of some era. Then, let X<sub>t</sub> be the total gas claims for some account's deploys in that era. Similarly, let Y<sub>t</sub> represent the total gas actually used within that era by the same account's deploys.

#### State updates

Suppose we are in the switch block of era t. *Current refund* due to the account is R<sub>t</sub> = X<sub>t</sub> - Y<sub>t</sub>.

The *accrued refund* **R**<sub>t</sub> represents the tokens yet to be disbursed. We update the accrued refund with current refund, **R**<sup>'</sup><sub>t</sub> = **R**<sub>t</sub> + R<sub>t</sub>.

The aggregate utilization **P**<sub>t</sub> is a weighted average of past refund percentages. Define the *current utilization* as P<sub>t</sub> = Y<sub>t</sub> / X<sub>t</sub>.

Calculate an updated aggregate utilization **P**<sup>'</sup><sub>t</sub> = (R<sub>t</sub>P<sub>t</sub> + **R**<sub>t</sub>**P**<sub>t</sub>) / **R**<sup>'</sup><sub>t</sub>.

We now disburse **R**<sup>'</sup><sub>t</sub>*f*(**P**<sup>'</sup><sub>t</sub>)<sup>-1</sup> tokens to the user account. Update accrued refund accordingly, **R**<sup>''</sup><sub>t</sub> = **R**<sup>'</sup><sub>t</sub>(1 - *f*(**P**<sup>'</sup><sub>t</sub>)<sup>-1</sup>).

In era t + 1, **R**<sub>t + 1</sub> = **R**<sup>''</sup><sub>t</sub> and **P**<sub>t + 1</sub> = **P**<sup>'</sup><sub>t</sub>.

### Account metadata in global state

The explicit account state and its updates described in the preceding section require persistent account metadata. It is suggested that this take form of a hash-indexed dictionary containing 3-tuples containing some numeric representation of the accrued refund, the numerator of the aggregate utilization and its denominator. It is expected that this dictionary would be visible under either the handle payment system contract or the mint system contract.

## Drawbacks

[drawbacks]: #drawbacks

### Account switching

Because refund metadata has to be attached to a particular account, users with lengthy disbursement schedules may attempt to evade the penalties by creating new accounts. However, this is not frictionless on the Casper platform, since new accounts require a minimum of 2.5 CSPR. Additionally, the penalties are temporary, since disbursement of all tokens due to be refunded amounts to a reset (the accrued refund becomes 0, giving 0 weight to past aggregate utilization).

### Low severity of the penalty

Although delayed disbursement should effectively disable a one-time attacker who congests the blockchain by overclaiming gas, eventually the attack can be repeated. This may be mitigated by introducing a limited form of incomplete refunds for very low aggregate utilization.

## Alternatives

[alternatives]: #alternatives

### Incomplete refunds

The platform already has the code to issue refunds, where percentage of the refund may vary anywhere between 0% (as it is set currently) and 100%. However, full refunds avoid compounding user frustration created by the gas estimation problem in the execution-after-consensus paradigm.


## Prior art

[prior-art]: #prior-art

No prior art is known.

## Future possibilities

[future-possibilities]: #future-possibilities

This feature is expected to be made even more attractive to users with the introduction of cost-regularizing gas discounts, to be described in a forthcoming CEP.