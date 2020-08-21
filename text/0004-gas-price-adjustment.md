# Daily Gas Price Adjustment Algorithm

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0004](https://github.com/casperlabs/ceps/pull/0004)

Gas price is regulated algorithmically. A price floor exists and adjusts daily, based on fullnesses of blocks from the previous day, in order to absorb volatility and reduce congestion.

## Motivation

[motivation]: #motivation

Gas prices are known to be subject to high volatility both in the short term and the long term. This is due to a combination of factors such as inelastic supply, [daily demand cycle](https://solmaz.io/2019/10/21/gas-price-fee-volatility/) and shifting trends.

During times of high demand for a blockchain, one can observe prices change up to 10-100x in a matter of hours. A floating gas price and the absence of a reference point exacerbates the situation. Such volatility makes it difficult for dapp providers and users to plan ahead their expenses.

We introduce a simple control algorithm in the spirit of [EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md), and aim to absorb majority of intraday volatility. Whereas EIP-1559 specifies block-by-block gas price adjustment, this proposal specifies a variant which does it day-by-day. The price floor adjusts at a relatively slower rate, i.e. 1-2% per day. This way, we aim for the platform to autonomously discover the price that fits the overall demand. By setting the right parameters, we aim to minimize congestions and keep prices at a stable level in the short term.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The algorithm sets a `BASE_PRICE` for every block, which the included transactions are charged by default. The paying signer accepts to be charged minimum `BASE_PRICE`, and must also specify a `MAX_GAS_PRICE` they are willing to accept as well. On top of `BASE_PRICE`, each transaction can specify a `PRICE_PREMIUM` which is added to `BASE_PRICE` to calculate the final gas price the transaction is charged. This is to enable users to prioritize their transactions if a congestion happens in spite of price adjustment.

`BASE_PRICE` starts at `INITIAL_BASE_PRICE` at genesis, remains constant for `ADJUSTMENT_PERIOD` ticks. Once that period is over, it can be updated once, in a system transaction. The process repeats at the end of each consecutive period of the same length.

The update can be either a percent increase or decrease of `BASE_PRICE`, specified by an `ADJUSTMENT_RATE`. To decide on the direction of the change, we make use of the fullnesses of blocks from the preceding adjustment period. We map the array of fullnesses to a single value called `FULLNESS` (details on how to compute this are given below). Then, if `FULLNESS` is greater than `TARGET_FULLNESS`, we increase the price. Otherwise, we decrease the price.

Explain the proposal as if it was already approved and implemented. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.

For implementation-oriented CEPs (e.g. for node internals), this section should focus on how other developers should think about the change, and give examples of its concrete impact. For policy CEPs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Protocol Variables

- `BASE_PRICE`: The value of the price floor, in motes. Given blocks proposed in a certain adjustment period, all transactions must be charged at least this value. The value will be included in each block in a field of the same name.
- `FULLNESS`: A value representing the fullness of blocks of a given adjustment period.

### Protocol Parameters

- `INITIAL_BASE_PRICE`: The value of `BASE_PRICE` at genesis. Value: TBD.
- `ADJUSTMENT_RATE`: The rate at which `BASE_PRICE` can move up or down per day. Value: `10_000_000_000` (1%).
- `DENOMINATOR`: Fractions will be represented as trillionths of 1. Value: `1_000_000_000_000`.
- `BLOCK_GAS_LIMIT`: Maximum gas allowed in a block. Value: TBD.
- `TARGET_FULLNESS`: The value which will determine whether `BASE_PRICE` should move up or down. Value: `650_000_000_000` (65%).
- `MIN_BASE_PRICE`: The minimum `BASE_PRICE` that is allowed for in the protocol. Value: TBD.

### Transaction Fields

- `PRICE_PREMIUM`: The price premium specified for a given transaction.
- `MAX_GAS_PRICE`: The maximum gas price which the paying signer is willing to accept. The maximum `BASE_PRICE` they are willing to accept is compted as `MAX_GAS_PRICE - PRICE_PREMIUM`.

### Transaction Variables

- `GAS_SPENT`: The total gas spent by a given transaction.

### Adjustment Process

For the `n`-th adjustment period, all finalized blocks with a timestamp in the range `[n*ADJUSTMENT_PERIOD, (n+1)*ADJUSTMENT_PERIOD]` are considered. The amount of gas used by each one is collected in an array `gas_used_array`. `FULLNESS` for the `n`-th period is computed as:

```
FULLNESS = DENOMINATOR * median(gas_used_array) / BLOCK_GAS_LIMIT
```

Then, a system transaction can be executed in any block proposed after `(n+1)*ADJUSTMENT_PERIOD`, which updates `BASE_PRICE` according to:

```
if FULLNESS > TARGET_FULLNESS:
    new_base_price = BASE_PRICE * (DENOMINATOR + ADJUSTMENT_RATE) / DENOMINATOR
else:
    new_base_price = BASE_PRICE * DENOMINATOR / (DENOMINATOR + ADJUSTMENT_RATE)

if new_base_price > MIN_BASE_PRICE:
    BASE_PRICE = new_base_price
```

This sets the new `BASE_PRICE` for the `n+1`-th adjustment period. The process repeats the same way for every adjustment period.

### Charging for Gas

For each transaction included in a block, the transaction's `MAX_GAS_PRICE - PRICE_PREMIUM` MUST be greater than the block's `BASE_PRICE`. The paying signer will end up paying

```
transaction_fee = (BASE_PRICE + PRICE_PREMIUM) * GAS_SPENT
```

## Drawbacks

[drawbacks]: #drawbacks

- The argument for this proposal favors *price stability* over *market efficiency*. In other words, we achieve stability by setting a price floor, and making users occasionally *pay higher* than the actual market price. If this argument is wrong, then the algorithm would be rendered useless.
- The adjustment rate may fall too short to adapt to short term trends, like sudden increased usage due to a hyped dapp.
- A scenario where majority of the validators coordinate and tamper with the price by filling up blocks with their own transactions. This would artifically increase `BASE_PRICE` and allow them to extract more profit from existing users. However, this type of behavior is also possible in the absence of price regulation, and the issue is orthogonal to the problem of volatility.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

As mentioned before, the closest precursor is [EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md), which is used for instantaneous price discovery, whereas this proposal aims to adjust at a much slower rate. As of writing this, EIP-1559 has not been implemented. Therefore, we can speculate about the potential success/failure of this proposal, but we cannot know for sure until we deploy it in production.

For the given problem, we didn't prefer EIP-1559 itself. That is because EIP-1559 is not designed to stabilize gas prices, but to enable objective and real time price discovery, and to increase fee market transparency and efficiency. Our goal was different from the very beginning.

Other types of control algorithms have already been used in many Proof of Work blockchains for difficulty adjustment. Bitcoin's market and miner ecosystem has faced many issues throughout its launch, however the protocol was always able to adapt to changing hashrate due to the robustness of the difficulty adjustment algorithm. For that reason, we have good reasons to believe that the proposed control algorithm will be as robust.

## Prior art

[prior-art]: #prior-art

As far as the author is informed, there has been no precedent to this type of price regulation, other than EIP-1559 which has not been used in production yet.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

TBD

## Future possibilities

[future-possibilities]: #future-possibilities

TBD
