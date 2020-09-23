# Gas futures

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0017](https://github.com/casperlabs/ceps/pull/0017)

A certain percentage of future block space is reserved and auctioned off. Said block space can only be filled by transactions signed by gas futures holders.

## Motivation

[motivation]: #motivation

Gas prices are known to be subject to high volatility both in the short term and the long term. This is due to a combination of factors such as inelastic supply, daily demand cycle and shifting trends. CasperLabs sees these issues with volatility as a key barrier to the adoption of public blockchain infrastructure by businesses. It is also recognized that price stability alone is not enough, and scalability is a core concern.

During times of high demand for a blockchain, one can observe prices change up to 10-100x in a matter of hours. A floating gas price and the absence of a reference point exacerbates the situation. Such volatility makes it difficult for dapp providers and users to budget for their expenses.

We propose gas futures to enable users to hedge themselves against future gas price uncertainty.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Prior to this proposal, each block had a `BLOCK_GAS_LIMIT` which determined the throughput of the blockchain. We introduce a protocol parameter `FUTURES_GAS_LIMIT <= BLOCK_GAS_LIMIT`. This portion of the block space will be exclusive to gas futures holders, that is

```python
X <= BLOCK_GAS_LIMIT - FUTURES_GAS_LIMIT
```

where `X` is the total gas usage of transactions signed by users that do not hold gas futures. Blocks that do not fulfill this condition are deemed invalid.

Gas futures are designated for consecutive periods of fixed length. In this proposal, we will focus on daily gas futures, but the same instrument can be extended to designate weekly, monthly, and so on. Gas futures are represented by tokens which are auctioned off continuously, some fixed duration preceding the futures date. (In this proposal, "futures" and "futures tokens" are used interchangeably.) These tokens will be designated in the format `GASYYYYMMDD`, and will be sold off in an on-chain auction `AUCTION_PERIOD` days before the futures date. In this proposal, we will assume `AUCTION_PERIOD = 180`, i.e. 6 months.

**Example:** Holders of the token `GAS20200901` will be able to include transactions on September 1, 2020 at the price they have agreed on at the auction. The auction takes place exactly 180 days before the delivery date, for the whole day, on March 5, 2020.

### Futures size

The protocol parameter `FUTURES_SIZE` determines how much gas a futures token corresponds to. In Casper, currently 1e12 block gas limit is envisioned. We take one-thousandth of a block to be a sufficiently small size for a futures contract. Therefore, we take `FUTURES_SIZE = 1_000_000_000`. Each futures token will grant the holder the right to include 1 billion gas worth of transactions.


### Futures quota

`FUTURES_GAS_LIMIT` varies for each day, depending on the amount of futures tokens sold off for that day. If no futures tokens were sold, it is simply equal to zero. There is an upper limit to the amount of futures tokens that can be sold at any given day, called `FUTURES_QUOTA`. We dictate to condition `FUTURES_SOLD[i] <= FUTURES_QUOTA` for any day `i`. Then `FUTURES_GAS_LIMIT` for `i` is determined by

```python
FUTURES_GAS_LIMIT[i] = BLOCK_GAS_LIMIT * FUTURES_SOLD[i] * FUTURES_SIZE / GAS_PER_DAY
```

**Example:** To determine `FUTURES_QUOTA`, we work our way back from the `FUTURES_GAS_LIMIT` we desire. Let us assume that we want 20% of block space to be reserved for futures transactions. This sets our upper limit in the case which the whole quota is sold off, i.e. `FUTURES_SOLD[i] = FUTURES_QUOTA`. Solving the equation with these values, we obtain `FUTURES_QUOTA = 0.2 * GAS_PER_DAY / FUTURES_SIZE`. Assuming `2**14 = 16384 ms` rounds, we have `60*60*24/16.384 ~ 5273` blocks per day, giving us `GAS_PER_DAY = 5.273e15`. Finally, we compute `FUTURES_QUOTA = 0.2 * 5.273e15 / 1e9 = 1054600` futures tokens per day.

### Auction

The auction for `GASYYYYMMDD` begins at tick `X - AUCTION_PERIOD * TICKS_IN_DAY` where `X` is the tick corresponding to `YYYY-MM-DD` 00:00 in UTC, and `TICKS_IN_DAY = 24*60*60*1000` is the number of ticks in one day. The auction can go on until the whole quota is sold off.

The auction price is determined by the validators collectively, who announce the price they are willing to accept. The futures price is computed by taking a weighted average:

```python
auction_price = sum(weight[v] * price[i] for v in validators) / sum(weight[v] for v in validators)
```

where `weight[v]` is the consensus weight of and `price[v]` is the price announced by validator `v`.

**Note:** `auction_price` designates the **gas price** that the user will receive on the day of delivery, not the price per futures token.

Preferred prices for a given futures can be announced prior to and during the auction by any validator. Auctions begin automatically without any outside intervention, and if a validator does not announce a price for the new futures, the price from the previous auction is assumed. Finally, the very first auction starts off with the preferred price set to zero for all validators.

### Payment and delivery

When a user buys a certain `amount` of the futures `GASYYYYMMDD`, they make an upfront payment of

```python
escrow = auction_price * amount * FUTURES_SIZE # CSPR
```

which is placed in escrow, until the date of delivery. This mints a [non-fungible token](https://en.wikipedia.org/wiki/Non-fungible_token) (NFT) which we call the *gas futures escrow token* (GFET). A GFET represents a user's ownership of the futures token, and stores the information

```python
(
    owner, # The user who bought the futures
    symbol, # Futures token symbol e.g. GAS20200901. Can also be any type of reference.
    amount, # The amount of futures bought
    price, # The auction price at the time of purchase
    total_gas_used, # Amount of gas used with this GFET
    escrow # Current amount of CSPR held in escrow
)
```

To fulfill the futures contract, a user has to include the GFET information in the transaction they submit on the day of delivery.

Let us assume that a user submits a transaction with a GFET, and it ends up using `gas_used <= amount * FUTURES_SIZE` gas.

- The user does not pay any additional fee, on top of the amount held in escrow.
- The validator who included the transaction receives `fee = gas_used * price`, paid from the escrow.
- Usage is deducted from the GFET, which is updated as:
  - `total_gas_used = total_gas_used + gas_used`.
  - `escrow = escrow - fee`.

This means that the same GFET can be used with multiple transactions, until it is fully depleted. Users always specify a gas price for payment at the spot price, in case the GFET gets depleted during execution. If that happens and the GFET cannot cover the whole transaction, the remainder is paid for regularly according to the spot price specified by the user. In that case, gas usage covered by the GFET is deducted from `BLOCK_GAS_LIMIT`, whereas the remainder is deducted from `BLOCK_GAS_LIMIT - GAS_FUTURES_LIMIT`. This solves the edge case related to the block validity condition we introduced above.

### Passing ownership

The GFET can be transferred to another account, upon which the `owner` field is set to the new owner. They can be transferred for free, or traded on an exchange. This way, users who missed out on auctions can still retroactively hedge themselves against gas price movements.

### Defaulted futures

When a GFET passes the delivery date without being fully used, it is said to be defaulted by the amount of `escrow` remaining in the GFET. Then it cannot be used anymore, and the only action left to do is to trigger the function `discount()` on it. This can be triggered by any account, once the delivery date passes. It does the following:

- The user receives `OWNER_DISCOUNT_RATE` percent of the remaining escrow.
- Validator set at the time of trigger receives `VALIDATOR_DISCOUNT_RATE` percent of the remaining escrow.
- The remaining escrowed CSPR is burned and the GFET is destroyed.

In this proposal, we assume both discount rates to be 20%. That means that 60% of remaining escrow gets burned.

## Drawbacks

[drawbacks]: #drawbacks

- Lack of enforcement
- Censorship
- Griefing

TBD.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

TBD.

## Prior art

[prior-art]: #prior-art

TBD.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

TBD.

## Future possibilities

[future-possibilities]: #future-possibilities

TBD.
