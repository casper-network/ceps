# Storage Token

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

Users need to own *storage tokens* (STR) to store data in the global state. The *STR contract* handles the minting and burning of STR. It is minted at a constant rate for each deposited CASPER. The same rate holds for redeeming and burning it. STR can be automatically minted during transaction execution, or it can be minted manually.

## Motivation

[motivation]: #motivation

The most straightforward way of pricing blockchain resources is to charge a fixed amount of gas per VM opcode, as a one-time payment. However, unlike computation which is used and freed immediately, persistent storage of data in the global state means that the resource continues to be used, unless it is freed manually by the user at a later time. For that reason, one-time payment is not the ideal model for pricing persistent storage.

To remedy this issue, Ethereum overprices storage, which curbs the growth rate of the global state. Ethereum also implements a *gas refund* mechanism, where users can discount gas usage of their transactions, if they free up previously used space. However, this arrangement does not ensure an upper bound to global state size. It grows at a linear rate, and one can see that it will require full nodes to have higher and more expensive storage[^1]. Unless an alternative model is implemented, the requirements could increase indefinitely, decreasing the number of full nodes due to increased costs. This can have adverse affects on the network.

Here, we propose an alternative model which leverages the scarcity of CASPER to limit the growth of global state, and increase the allocative efficiency of storage.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

We introduce the concept of *data ownership*: every bit of data stored in the global state is associated with an account. Accounts must hold storage tokens in order to store data, denoted internally by the symbol STR. STR is indivisible and is the smallest denomination of itself. 1 STR gives its holder the right to store 1 byte of data.

There exists a STR contract which handles minting and burning of STR. STR can be minted automatically, during transaction execution, or manually, when a user calls the contract. In both cases, CASPER must be deposited by the associated account. The STR contract retains a constant `STR_PRICE`, set at genesis, to be updated only through governance. Whenever `X` STR is to be minted, the receiving account is required to deposit `X * STR_PRICE` CASPER in the STR contract.

In each transaction, it is made clear which account would be charged for STR. Each transaction has mandatory fields `GAS_LIMIT` and `STR_LIMIT`, so that the transaction is reverted if usage of either surpasses the corresponding limit. The same holds if the account does not hold enough CASPER to mint the required STR. Storage is metered as the transaction gets executed, so that it can halt immediately when any of these conditions is not satisfied.

We keep track of the actual amount of data an account stores in the global state at any time. An account can try to redeem `X` STR at the current `STR_PRICE`, but it can succeed only if the difference between the STR held by the account and the number of bytes stored by it is greater than or equal to `X`.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

TBD

## Drawbacks

[drawbacks]: #drawbacks

- There is a possbility that the overhead of STR accounting ends up outweighing the benefits (allocative efficiency, guaranteed limit on storage requirements).
- The fixed rate collateralizing of CASPER for STR limits the changes that can be made to `STR_PRICE`. Specifically, it *can be decreased* by paying out `(STR_PRICE_OLD - STR_PRICE_NEW) * STR_SUPPLY` CASPER pro rata to existing holders. However, it *cannot be increased*, since that would cause STR to be undercollateralized.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

TBD

## Prior art

[prior-art]: #prior-art

TBD

## Unresolved questions

[unresolved-questions]: #unresolved-questions

TBD

## Future possibilities

[future-possibilities]: #future-possibilities

TBD

[^1]: Full node storage requirements can be viewed on [Etherscan](https://etherscan.io/chartsync/chaindefault). For Geth, it grew 297 GB between 2019-08-27 (172 GB) and 2020-08-27 (469 GB). At this rate, it would surpass 1 TB by the end of 2022.
