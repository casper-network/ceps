# Storage Token

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0011](https://github.com/casperlabs/ceps/pull/0011)

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


As mentionioned in [Motivation](#motivation), Ethereum implements the one-time payment model for storage. This is as simple as it gets. However, blockchain storage space is a common good, and users should not be able to occupy it indefinitely, for free. The gas refund mechanism provides incentives for freeing up, but it doesn't solve the problem of perpetual growth of global state.

To facilitate a more efficient allocation of storage, we came up with two alternatives:

- **Storage rent:** Users continuously pay a fixed fee per byte stored in the global state. This creates direct incentives for deleting redundant data. However, it is hard to design for edge cases, such as what would happen if an account defaults on a payment. It would probably call for a deletion, equivalent to the defaulted amount. But one can never determine fairly which part of a user's data should be deleted, due to how storage works.
- **Storage token:** This proposal, which brings the benefits of the gas refund mechanism, but also leverages scarcity of CASPER to solve the problem of perpetual growth. Also, gas refund only allows for redeeming storage space for gas, which is to be used in that instant. However with storage token, storage space is redeemed for CASPER, which can be kept for other purposes.

Due to issues with edge cases and problematic UX of storage rent, we decided to go with storage token.


## Prior art

[prior-art]: #prior-art

Ethereum's one-time payment and gas refund mechanism has already been explained in the previous sections. We use this section to mention [GasToken](https://gastoken.io/), a token on Ethereum which exploits the refund mechanism in order to get a cheaper gas price. It provides users a streamlined way of using up storage, so that it can be redeemed at a later point for cheaper gas. Although it gives the illusion of "stored gas", it actually takes advantage of the inefficiency of the gas refund mechanism. For that reason, there is a limit to the amount that can be saved by using it.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

TBD

## Future possibilities

[future-possibilities]: #future-possibilities

TBD

[^1]: Full node storage requirements can be viewed on [Etherscan](https://etherscan.io/chartsync/chaindefault). For Geth, it grew 297 GB between 2019-08-27 (172 GB) and 2020-08-27 (469 GB). At this rate, it would surpass 1 TB by the end of 2022.