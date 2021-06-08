# Signature Based Finality

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

This document is a follow-up to [a point in CEP 36](https://github.com/casper-network/ceps/blob/master/text/0036-consensus-memory-usage.md#finality-is-determined-by-finality-signatures-only), specifying the details of how block finality based on signatures should be implemented.

## Motivation

[motivation]: #motivation

The consensus component is currently quite heavy on memory, having to store the full protocol states of multiple eras from the past, in case some nodes decide to equivocate in those eras and have to be punished. Also, communicating the protocol state to other nodes is currently the only way to make them finalize blocks and advance to later eras if they are lagging behind. Thus, if we want to be able to drop older eras, we have to make sure that:

- we can detect attempts to create a fork in the chain by equivocating, and
- we can prove finality of the latest blocks to lagging nodes.

Both of these goals can be achieved with signature based finality.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The current process of block finalization is this:

1. A `BlockPayload` gets finalized by a consensus instance.
2. The consensus component announces a `FinalizedBlock`.
3. Contract runtime executes the block and announces a `Block`.
4. Linear chain component stores the block and announces that it was stored.
5. Consensus creates a finality signature for the block.
6. Linear chain stores the signature and gossips it.

In order to only consider fully signed blocks as finalized, we'll have to change this process to the following:

1. A `BlockPayload` gets finalized by a consensus instance.
2. The consensus component announces a `FinalizedBlock`.
3. Contract runtime executes the block and announces a `Block`.
4. In response to the announcement, linear chain component caches the block, and consensus creates a finality signature.
5. Linear chain stores the signature and gossips it. It also receives signatures sent over the network.
6. Once the linear chain gathers enough signatures for the block, it considers it finalized.
    - If the block is unknown yet, it requests the block from the sender of the last signature.
    - If the block is known, but it isn't the immediate descendant of the highest block so far, it requests the range of blocks from the immediate descendant of the highest block to the just finalized block.
    - If the block is known and it is the immediate descendant of the highest block so far, it adds the block to the chain as the new highest block.

A node receiving the request sent in point 6 would respond with the following:
- To the request for a single fully signed block, send the block along with signatures proving it is fully signed.
- To the request for a range of fully signed blocks, send the fully signed blocks one by one, starting from the oldest one.

Upon receiving a block with signatures as a response:
    - If it's not fully signed, discard and re-request it from someone else.
    - If it's the immediate descendant of the highest known block:
        - Request execution by the contract runtime.
        - Compare the execution results to the block - if they don't match, discard the block and re-request it from someone else.
        - If everything fits, store the block as the new highest block.
    - If it's not the immediate descendant of the highest known block, request the range of blocks from the immediate descendant of the highest block to this one.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The `ContractRuntimeAnnouncement::LinearChainBlock` has to be handled both by the `LinearChain` and consensus.

`LinearChain`, instead of storing it immediately, would first cache it and wait for finality signatures. Only after it is fully signed and the immediate ancestor of the highest finalized block, will it be appended to the linear chain and announced in a `BlockAdded` announcement.

Consensus would react to the `LinearChainBlock` announcement the way it currently handles a `BlockAdded` announcement - it would create and announce a finality signature.

The `ConsensusAnnouncement::CreatedFinalitySignature` would be handled by the linear chain as before, except now the `LinearChain` component would check whether the signed block is now fully signed. If it is, it would do the checks described in the sub-points of point 6 in the [Guide level explanation](#guide-level-explanation).

Two new network message variants would be added to `protocol::Message`: `RequestBlockRange` and `FullySignedBlock`.

Whenever the `LinearChain` component collects enough finality signatures for a block hash, but it doesn't know the block, it needs to request that block. It can be done with the already existing `Message::GetRequest`.

`RequestBlockRange(a, b)` handles the use case when a new fully signed block is not the immediate descendant of the highest known fully signed block. `a` and `b` denote the minimum and maximum requested block heights.

`FullySignedBlock(block, signatures)` would be used as a response to a `RequestBlockRange`. Sometimes, multiple such responses would be required.

## Drawbacks

[drawbacks]: #drawbacks

This complicates the process of finalization slightly, and will probably increase time to finality (the time elapsed between when the block was proposed and when it was fully finalized, ie. fully signed). However, the significant memory savings enabled by this are worth the tradeoff.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

In order not to sacrifice security, we have to be able to detect equivocations. This is what caused us to keep a significant number of old eras in memory, causing the large memory footprint of consensus. This design enables reducing it with no adverse effects to security: DAG equivocations in old eras become irrelevant, as they can't cause trouble themselves, because a block will not be considered finalized until it is fully signed. Making two conflicting blocks fully signed, on the other hand, requires creating equivocating finality signatures, which can be detected and penalized.

Because the ability to detect potentially harmful equivocations is a hard requirement, there doesn't seem to be any alternative that would not sacrifice at least some of the security properties of the protocol.

## Prior art

[prior-art]: #prior-art

None at the moment.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

None at the moment.

## Future possibilities

[future-possibilities]: #future-possibilities

This CEP is designed to enable memory optimizations in consensus, so that's the main future possibility.

The next step to achieve that would be to respond with a fully signed switch block of era `N` to any message in era `N`, whenever the most recent era number is greater than `N`. This should prompt the recipient to download all the blocks they are missing and catch up with the most recent era quickly.

Then, the final step would be to drop an era whenever we have a fully signed switch block in that era.

We should also plan adding detection of equivocating finality signatures (finality signatures by the same author of two blocks with the same height, but different hashes).
