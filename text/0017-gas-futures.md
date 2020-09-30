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

Prior to this proposal, each block had a `BLOCK_GAS_LIMIT` which determined the throughput of the blockchain. We introduce a protocol parameter `MAX_FUTURES_GAS_LIMIT <= BLOCK_GAS_LIMIT`. This portion of the block space will be exclusive to gas futures holders, that is

```python
X <= BLOCK_GAS_LIMIT - FUTURES_GAS_LIMIT[i]
```

where `X` is the total gas usage of transactions signed by users that do not hold gas futures, and `FUTURES_GAS_LIMIT[i]` is the gas limit for futures transactions in block `i`. Blocks that do not fulfill this condition are deemed invalid. We explain how `FUTURES_GAS_LIMIT[i]` is computed below.

Gas futures allow users to reserve block space at a certain block height. Block space is auctioned off continuously. Following a purchase, an NFT called *futures escrow token* is minted, whose owner can get their transactions included at the predetermined futures price. Futures prices are determined during the on-chain auction, which takes place `GAS_AUCTION_PERIOD` days before the futures date. In this proposal, we will assume `GAS_AUCTION_PERIOD = 180`, i.e. 6 months.

We tokenize block space, and denote it by the symbol BST, which stands for *block space token*. The reservation is specified in terms of a given day and block height, designated as `BST-YYYY-MM-DD-NNNNN`, where `YYYY-MM-DD` denotes the date in ISO-8601 format and `NNNNN` is the height of the block in that day. Specifically, the first block proposed at or after the first tick of that day is assigned the height 0, and its children are assigned 1, 2 and so on.

**Example:** Holders of the token `BST-2020-09-01-01000` will be able to include transactions in the 1000th block that is proposed on September 1, 2020 at the price they have agreed on at the auction. The auction takes place exactly 180 days before the delivery date, for the whole day, on March 5, 2020.

### Futures size

The protocol parameter `BST_SIZE` determines how much gas a BST corresponds to. In Casper, currently 1e12 block gas limit is envisioned. We take one-thousandth of a block to be a sufficiently small size for a futures contract. Therefore, we take `BST_SIZE = 1_000_000_000`. Each BST will grant the holder the right to include 1 billion gas worth of transactions.

The maximum supply of a BST token for a given block is found by `MAX_FUTURES_GAS_LIMIT / BST_SIZE`.

### Futures gas limit

`FUTURES_GAS_LIMIT` varies for each block, depending on the amount of BST sold off for that block. If no BST were sold, it is simply equal to zero. We dictate `BST_SOLD[i] <= MAX_FUTURES_GAS_LIMIT / BST_SIZE` for any block `i`. Then `FUTURES_GAS_LIMIT` for `i` is determined by

```python
FUTURES_GAS_LIMIT[i] = BST_SOLD[i] * BST_SIZE
```

<!-- ### Futures quota -->

<!-- `FUTURES_GAS_LIMIT` varies for each day, depending on the amount of BST sold off for that day. If no BST were sold, it is simply equal to zero. There is an upper limit to the amount of BST that can be sold at any given day, called `FUTURES_QUOTA`. We dictate to condition `FUTURES_SOLD[i] <= FUTURES_QUOTA` for any day `i`. Then `FUTURES_GAS_LIMIT` for `i` is determined by -->

<!-- ```python -->
<!-- FUTURES_GAS_LIMIT[i] = BLOCK_GAS_LIMIT * FUTURES_SOLD[i] * BST_SIZE / GAS_PER_DAY -->
<!-- ``` -->

<!-- **Example:** To determine `FUTURES_QUOTA`, we work our way back from the `FUTURES_GAS_LIMIT` we desire. Let us assume that we want 20% of block space to be reserved for futures transactions. This sets our upper limit in the case which the whole quota is sold off, i.e. `FUTURES_SOLD[i] = FUTURES_QUOTA`. Solving the equation with these values, we obtain `FUTURES_QUOTA = 0.2 * GAS_PER_DAY / BST_SIZE`. Assuming `2**14 = 16384 ms` rounds, we have `60*60*24/16.384 ~ 5273` blocks per day, giving us `GAS_PER_DAY = 5.273e15`. Finally, we compute `FUTURES_QUOTA = 0.2 * 5.273e15 / 1e9 = 1054600` BST per day. -->

### Auction

The auction for `GAS-YYYY-MM-DD-NNNNN` begins at tick `X - GAS_AUCTION_PERIOD * TICKS_IN_DAY` where `X` is the tick corresponding to `YYYY-MM-DD` 00:00 in UTC, and `TICKS_IN_DAY = 24*60*60*1000` is the number of ticks in one day. The auction can go on until the whole quota is sold off.

The auction price is determined by the validators collectively, who announce the price they are willing to accept. The futures price is computed by taking a weighted average:

```python
auction_price = sum(weight[v] * price[i] for v in validators) / sum(weight[v] for v in validators)
```

where `weight[v]` is the consensus weight of and `price[v]` is the price announced by validator `v`.

**Note:** `auction_price` designates the **gas price** that the user will receive on the day of delivery, not the price per BST.

Preferred prices for a given futures can be announced prior to and during the auction by any validator. Auctions begin automatically without any outside intervention, and if a validator does not announce a price for the new futures, the price from the previous auction is assumed. Finally, the very first auction starts off with the preferred price set to zero for all validators.

### Payment and delivery

When a user buys a certain `amount` of the futures `BST-YYYY-MM-DD-NNNNN`, they make an upfront payment of

```python
escrow = auction_price * amount * BST_SIZE # CSPR
```

which is placed in escrow, until the date of delivery. This mints a [non-fungible token](https://en.wikipedia.org/wiki/Non-fungible_token) (NFT) which we call the *futures escrow token* (FET). A FET represents a user's ownership of the BST, and stores the information

```python
(
    owner, # The user who bought the futures
    symbol, # BST symbol e.g. BST-2020-09-01-01000. Can also be any type of reference.
    amount, # The amount of futures bought
    price, # The auction price at the time of purchase
    total_gas_used, # Amount of gas used with this FET
    escrow # Current amount of CSPR held in escrow
)
```

To fulfill the futures contract, a user has to include the FET information in the transaction they submit on the day of delivery.

Let us assume that a user submits a transaction with a FET, and it ends up using `gas_used <= amount * BST_SIZE` gas.

- The user does not pay any additional fee, on top of the amount held in escrow.
- The validator who included the transaction receives `fee = gas_used * price`, paid from the escrow.
- Usage is deducted from the FET, which is updated as:
  - `total_gas_used = total_gas_used + gas_used`.
  - `escrow = escrow - fee`.

This means that the same FET can be used with multiple transactions, until it is fully depleted. Users always specify a gas price for payment at the spot price, in case the FET gets depleted during execution. If that happens and the FET cannot cover the whole transaction, the remainder is paid for regularly according to the spot price specified by the user. In that case, gas usage covered by the FET is deducted from `BLOCK_GAS_LIMIT`, whereas the remainder is deducted from `BLOCK_GAS_LIMIT - GAS_FUTURES_LIMIT`. This solves the edge case related to the block validity condition we introduced above.

### Passing ownership

The FET can be transferred to another account, upon which the `owner` field is set to the new owner. They can be transferred for free, or traded on an exchange. This way, users who missed out on auctions can still retroactively hedge themselves against gas price movements.

### Defaulted futures

When a FET passes the delivery date without being fully used, it is said to be defaulted by the amount of `escrow` remaining in the FET. Then it cannot be used anymore, and the only action left to do is to trigger the function `discount()` on it. This can be triggered by any account, once the delivery date passes. It does the following:

- The user receives `OWNER_REFUND_RATE` percent of the remaining escrow.
- Validator set at the time of trigger receives `VALIDATOR_DISCOUNT_RATE` percent of the remaining escrow.
- The remaining escrowed CSPR is burned and the FET is destroyed.

In this proposal, we assume both discount rates to be 20%. That means that 60% of remaining escrow gets burned.

### Overdrifted futures

Gas is reserved at a specific block height in a given day, so the delivery date can drift to a later time, if there is a liveness fault. If a block cannot be proposed before the end of the specified date, it is said to be *overdrifted*. The gas futures associated with an overdrifted block overlap with the block from the following day, and thus cannot be fulfilled. We treat this case differently than defaulted futures, because it is not necessarily the user's or the specific validator's fault. Thus, in the case of overdrifted futures, the user can withdraw 100% of the escrow.

## Drawbacks

[drawbacks]: #drawbacks

- **Lack of enforcement:** Futures agreements cannot be legally enforced on chain, so the only way to ensure validators and users fulfill them is to use incentives.
- **Censorship:** A validator could censor a transaction with gas futures, if it were in their best interest. There is no way to prevent this.
- **Griefing:** If defaulting incurred penalties on either side, either side could use this to grief the other side.


## Future possibilities

[future-possibilities]: #future-possibilities

- **Gas futures with lower time-specificities:** For example, reserve `X` gas to be used any time in a e.g. 1 hour range, instead of a specific block. One could even think of the general case of reserving `X` gas to be used between block heights `A` and `B`. In that case, the auction mechanism would need to ensure that the amount of futures sold does not cause the futures gas limit to be exceeded for any block. The most general version seems to admit some sort of [packing problem](https://en.wikipedia.org/wiki/Packing_problems). If the solution to this problem proves to be impractical, we could imagine having limited time specificities, e.g. per-block-height and per-hour. Then, block space could be reserved separately for different time specificities, where each has its own gas limit, i.e. `PER_BLOCK_HEIGHT_FUTURES_GAS_LIMIT` and `PER_HOUR_FUTURES_GAS_LIMIT`. This would make it easier to enforce block validity conditions.

