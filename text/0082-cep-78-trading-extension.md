# CEP-82 - Trading Extension for CEP-78

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#82](https://github.com/casperlabs/ceps/pull/82)

Trading capabilities for CEP-78 based NFTs and associated conventions.

## Motivation

[motivation]: #motivation

The goal of this CEP is to explore the current capabilities of CEP-78 to facilitate NFT trading use cases, and to outline flows and interfaces that could be standardized for greater interoperability of different marketplace implementations and CEP-78 NFTs.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

We introduce a new contract type called a `Marketplace`, the main purpose of which is to facilitate the exchange of NFTs between interested parties. There are two primary cases to consider:
* Exchanging NFTs for other NFTs,
* Exchanging NFTs for fungible tokens.

### Custody

The `Marketplace` contract can control the exchanged NFTs by either:
* Having ownership of the NFT transferred to it directly,
* Becoming an `operator` of the NFT via approval.

Which one is more appropriate will depend on the marketplace and its model, and can be controlled through a _modality_ on the contract. The presence of this modality is also important for informing the consumers of the `Marketplace` contract on which mechanism to use.

* `CustodyApproval` - custody via approval
* `CustodyTransfer` - custody via transfer

Taking ownership of the NFT is needed in cases where it is necessary to prevent the owner of the NFT from unexpectedly transferring the NFT to a third party, such as during an auction. Otherwise, approvals are more appropriate, as they do not change the actual owner of the NFT. 

_(wip) remark: this requirement might conflict with the potential implementation of forced royalty collection using an intermediary custodial/royalties contract_

### Duration

Another dimension that can differentiate `Marketplace` contracts is _duration_:
* A `Marketplace` can be _instant_, meaning that an exchange of items can happen as soon as two parties agree, such as in a swap. A consequence of this is that only the _seller_ needs to hand over custody of the item being sold, whereas the _buyer_ operation can be atomic. `CustodyApproval` will suffice for this `Marketplace`
* A `Marketplace` can be _continuous_ (find a better word?), meaning that an exchange can happen only when a duration of time has elapsed or multiple parties have participated, such as in an auction. In general, this will require that each party give the marketplace full custody of the asset being traded, as trying to handle the edge cases where an asset disappears from a running auction would introduce unnecessary complexity. `CustodyTransfer` is required for this `Marketplace`.

Similarly, this introduces two distinct modalities for a `Marketplace`: 
* `DurationInstant`
* `DurationContinuous`

### Interface / Key Functions

Only the API surface that would be common between all reasonable `Marketplace` implementations can be clearly defined. Key points:

* The entry point for _selling_ is a `post` method,
* The entry point for _buying_ is a `bid` method, 
* Every created posting / trade gets assigned a unique `trade_id`, that is returned from the corresponding `post` method,
* Posted trades can be cancelled by the trade poster via a `cancel` method, if applicable,
* Placed bids can be retracted via an `unbid` method, if applicable,

For `Marketplace`s of `CustodyTransfer` modality:
* `get_real_owner` method for querying the "real" owner for an NFT in custody,

The `post` and `bid` methods are _conventions_ as opposed to a strict interface. Different implementations of `Marketplace` can have different requirements and user flows as necessitated by their logic, and forcing them to conform to a single API could unnecessarily restrict them.

### Initial Reference Implementation

The reference implementation includes the above modalities, and allows the contract instance to configure the accepted NFT and FT kinds. It supports the following models:
* An "order book" type of marketplace for NFT-FT exchanges. Sellers create a posting for their NFT and an asking price. Buyers can acquire NFTs by matching the asking price, and the `Marketplace` contract performs an atomic swap of assets.
* An "auction house" type of marketplace for NFT-FT exchanges. Sellers create a posting for their NFT and an initial price. Buyers bids on the NFT until a certain deadline (specified by the seller, up to a reasonable limit), after which all bidding operations are frozen. The seller and the auction winner are then allowed to trigger a finalization, which will perform an atomic asset exchange. Neither is allowed to unbid or cancel the trade at this point. 

## Reference-level explanation
 
[reference-level-explanation]: #reference-level-explanation

// TODO

## Drawbacks

[drawbacks]: #drawbacks

// TODO

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

### API Surface

While it is possible to design for a variety of `Marketplace` implementations ahead of time, this could inadvertently make certain implementations artificially inviable or challenging, only because of the requirement to conform to a specific API. Because of this, we try to take a more relaxed approach, defining the API surface for functionality that is _guaranteed to be shared_ between _all or most_ `Marketplace` implementations, while leaving the more variable parts up to the implementation or future revisions of this CEP.

This doesn't mean that specific, rigid API surfaces can't be defined for certain kinds of `Marketplace` implementations in the future, however. 

## Prior art

[prior-art]: #prior-art

// TODO

## Unresolved questions

[unresolved-questions]: #unresolved-questions

* What other marketplace types would be useful in the reference implementation?
* Should NFT-NFT marketplace implementations be included in the reference implementation?
* How do royalties fit into this picture? 

## Future possibilities

[future-possibilities]: #future-possibilities

A definite next step after this draft is reviewed and improved is the inclusion of royalties mechanics, thus expanding this from a simple "trading" CEP to a "trading & royalties" CEP.

If this CEP is accepted, next steps could also be:
* Defining concrete API surfaces for certain `Marketplace` schemes,
* Adding more marketplace models to the reference implementation,
