# Block Proposer Improvements

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0020](https://github.com/casperlabs/ceps/pull/0020)

This CEP describes suggested modifications to the Block Proposer component designed to fix some of the known problems with it.

## Motivation

[motivation]: #motivation

There are a few known problems in the Block Proposer related to event ordering. The component is written with the assumption that it will receive events in the same order they were generated, but because of the asynchronous nature of event handling within the reactor, this assumption is sometimes incorrect.

There are two main ways in which a violation of the underlying assumptions can cause issues with the Block Proposer:

- it can see blocks being finalized before it sees them proposed
- it can be asked to propose a block before it receives a notification about a block being proposed by another validator, leading to duplications of deploys.

A separate solution for each of these two issues will be described below.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

### Seeing finalization before proposal

The suggested solution would be to not notify the Block Proposer about proposed blocks at all. The `past` parameter passed to it would contain the hashes of deploys contained in the blocks that haven't been finalized yet, instead of the hashes of blocks themselves. This would make the `past` list a few times longer, but would remove the need for the Block Proposer to know which blocks have been proposed - it could only care about finalized blocks in this approach.

One of this solution's advantages would be removing the need for detection of orphaned blocks. If we don't record proposals, we won't have to "restore" the deploys from there when a block is orphaned.

Another advantage over the main alternative discussed below is the lack of a need to distinguish between identical deploy lists. We wouldn't care if multiple proto-blocks with empty lists have been proposed, we would only care about the actual set of deploys that are in the past of the block to be proposed.

### Request for a block before seeing another validator's proposal

We have some measures to prevent deploy replays in place: when requested for a set of deploys to propose, the Block Proposer should also receive data about the new block's past, that is, the hashes of blocks that will be in the block's past, but aren't yet finalized. The Block Proposer itself is supposed to be aware of all the finalized blocks, which will also constitute the new block's past because of what finalization means.

However, the issue that can arise, is that Consensus issues a notification about finalization of a block, and a request for deploys shortly afterwards (so that it doesn't include the newly finalized block in the information about the past of the block to be proposed), but the Block Proposer receives these events in reverse order, thus making it not aware that the newly finalized block should also be considered a part of the new block's past.

The solution for this issue could be to include the hash of the last finalized block in the request for deploys. The Block Proposer could then check if it already has that block as finalized, and if not, it would buffer the request and complete it when it receives the notification of the relevant block's finalization.

Even with this solution in place, we could actually still experience problems. Consensus could finalize multiple blocks at once, and if the events aren't guaranteed to arrive in order, the Block Proposer could be notified of the finalization of the last block before some others. In such a case, it would still have an incomplete picture of the new block's past. To prevent this, we would have to only handle notifications about finalized blocks if the block's parent has already been finalized (and buffer finalization notifications that don't satisfy this condition, until it is satisfied).

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Seeing finalization before proposal

- Remove `self.state.proposed`, `added_block()`, `orphaned_block()`, `Event::ProposedProtoBlock` and consensus announcement about a proposed proto block
- In `remaining_deploys`, change `past_blocks` to `past_deploys` which would be a set of deploy hashes instead of block hashes
- Change the calculation of `past_deploys` in the method to append the deploys from finalized blocks to `past_deploys`

### Request for a block before seeing another validator's proposal

This would require adding a `last_finalized_block: BlockHash` field to `BlockProposerRequest::ListForInclusion`.

When receiving a `ListForInclusion` request:

- check if `last_finalized_block` is in `self.state.finalized`
- if yes, proceed in the current manner
- if not, buffer the request in a new field `self.state.request_queue`

When receiving a `FinalizedProtoBlock` event:

- check if the block's parent is in `self.state.finalized` (in addition to other checks that are in place)
- if yes, proceed; re-check whether blocks in `self.state.finalization_queue` can be finalized and finalize those that can
- if no, buffer the block in `self.state.finalization_queue`

## Drawbacks

[drawbacks]: #drawbacks

This increases the size of the parameters passed around when requesting a new block from the Block Proposer. It also adds some state (`finalization_queue`) and fields to events (`last_finalized_block`). However, the issues caused by out-of-order events are pressing and the benefits likely outweigh the cost.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The main alternative to the first issue would be to buffer finalized blocks that haven't been proposed as far as we know, and then handle both proposal and finalization when we see the proposal.

Since both the proposal and finalization events ultimately come from our own Consensus component, and we won't finalize a block we haven't seen proposed, we could expect that when the Block Proposer sees a finalization of a block that apparently wasn't proposed, it will also see the proposal shortly. This means we wouldn't even have to take special care to prune old finalized blocks from the buffer, since we should never see a block that has been finalized, but is never proposed.

However, a disadvantage here relative to the suggested solution is the issue of orphaned blocks. Since we have no orphaned block detection currently, a block that is orphaned will stay in the `proposed` collection forever, thus making nodes never propose the deploys that were contained in them.

Another pain point would be that we would need a way to tell proto-blocks with identical deploy lists (like an empty list) from each other - otherwise, when we receive a notification about a block being finalized, we wouldn't know which proto-block it refers to, and we wouldn't be able to have multiple identical proto-blocks in the `proposed` collection. A way of doing that would be to pass in a parent block hash along with the proto-block.

Another alternative for the first issue might be requiring consensus to wait with issuing a finalization event until it is sure that the proposal event has been processed by all relevant components. This would just move the additional required state from Block Proposer to Consensus, though, and would require a much greater complexity in the event flow (announcements would have to be responded to with confirmations, which would have to be processed).

An alternative for the second issue could be to make sure that replaying a deploy won't cause any problems. This could be achieved by making all deploys idempotent somehow, but there is no clear way of making them so, making this possible solution purely hypothetical.

## Prior art

[prior-art]: #prior-art

Unknown. This issue is rather specific to our node's architecture.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

None at the moment.

## Future possibilities

[future-possibilities]: #future-possibilities

The solution to the first issue could be later optimized by restoring the notifications about proposed blocks, but they would only be used as a means to make the `past` list shorter - a hash of a proto-block would serve as an abbreviation for a list of deploys.
