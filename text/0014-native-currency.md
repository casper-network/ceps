# Native Currency of the Casper Network

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0014](https://github.com/casperlabs/ceps/pull/0014)

The Casper Network has a native currency, called Casper, and denoted by the ticker symbol CSPR.

## Motivation

[motivation]: #motivation

To create a reference for CSPR and how it is denominated.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The Casper Network has a native currency, called Casper, and denoted by the ticker symbol CSPR. It is primarily used to pay for transaction fees, but can also serve many other purposes, such as a store of value, collateral, or any function users see fit.

The smallest subdenomination of CSPR is called *mote*. We set 1 CSPR = 10^9 mote. We denote greater subdenominations with SI prefixes:

| Multiplier | Name |
|-|-|
| 10^0 | mote |
| 10^3 | kilomote |
| 10^6 | megamote |
| 10^9 | gigamote = Casper |

We represent all Casper related quantities in terms of motes at the implementation level. The number of decimals is chosen so that we will never need to subdivide the smallest subdenomination.

The initial supply of Casper at mainnet launch will be 10 billion. The issuance schedule is out of the scope of this proposal.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

N/A.

## Drawbacks

[drawbacks]: #drawbacks

N/A.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

We have two goals when we set how Casper is denominated:

- The circulating supply of the base unit, i.e. mote, must be large enough so that we will never need to subdivide it.
- The accumulation of round-off errors is minimized.

We tried to reason according to these goals for our decision, and made comparisons with other blockchains.

## Prior art

[prior-art]: #prior-art

We define `total_supply` as the total supply of a blockchain's native currency, and `decimals` as the number of decimal places representing its smallest subdenomination.

Then supply of the base unit is computed as `base_supply = total_supply * 10^decimals`.

| Blockchain | Total supply | Decimals | Base supply |
|-|-|-|-|
| Bitcoin | 21 million | 8 | ~10^15.32 |
| Ethereum | ~100 million | 18 | ~10^26 |
| Polkadot | ~1 billion | 10 | ~10^19 |
| Casper | 10 billion | 9 | 10^19 |

The benchmark for base supply is determined by the following condition:

```
base_supply >= token_market_cap * 100
```

where `token_market_cap` is in fiat terms. If this condition is not satisfied, the blockchain native currency becomes unable to denominate the smallest fiat subdenomination, e.g. cents of USD. For Bitcoin, this would happen at 21 trillion USD market cap, where 1 satoshi would equal 1 cent. Although this seems like an unlikely scenario, blockchains are launched with grandiose goals and can persist for a long amount of time. So it makes sense to set platform parameters considering what might happen at least in the next 100 years.

Ethereum's developers were probably aware of this, so they launched it with a higher base supply, roughly 10 billion times that of Bitcoin's. Gavin Wood, the author of the Ethereum yellow paper and possibly the original source of this idea must have realized that it is a bit of an overkill, so he launched his next project, Polkadot, with a lower base supply, roughly 10 thousand times that of Bitcoin's. In this proposal, we apply a similar rationale and choose 9 decimals, giving us the same base supply. Moreover, 1 Casper conveniently coincides 1 billion motes or 1 gigamotes, which is more intuitive and easy to remember.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

N/A.

## Future possibilities

[future-possibilities]: #future-possibilities

N/A.
