# Archival Nodes

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

This CEP outlines the design of archival nodes, that is, nodes keeping the complete history of the Casper blockchain.

## Motivation

[motivation]: #motivation

The Casper blockchain, like all blockchains, is rapidly growing, and requires more and more storage space to be kept in its entirety. This is very wasteful, as only a small number of most recent blocks are strictly required for regular operations. It is thus tempting to simply discard the blocks that are no longer relevant, but this would mean losing almost all of the network's history, which could turn out to be catastrophic if something goes wrong, and would go against the idea of a blockchain being a public ledger of all operations ever executed on it.

There is a middle ground here, though - instead of having all nodes keep the whole history, or having all nodes discard older blocks, we could have most of the nodes only keeping what is relevant for day-to-day operations, and only a small number of nodes keeping the complete history of the network. These history keepers would be called the "archival nodes".

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Until Fast Sync is introduced, all nodes are effectively archival nodes, as every time a node joins the network, it has to sync the whole blockchain since genesis. Thus, once it becomes able to participate in the network, it already has all the history stored locally.

Fast Sync is going to change that picture. Once it starts being the method used for syncing, the joining nodes will only download some of the switch block headers and the global state from a recent block, ignoring a huge part of the network's history. This means that if a fast syncing node didn't have all the history already, it won't have at least some of it afterwards, either.

Since Fast Sync has to be able to download full data of at least a single block, we propose for the archival nodes to use Fast Sync capabilities to sync block by block, from genesis up to the most recent blocks.

The process will be analogous to the pre-fast-sync Linear Chain Sync:

1. Walk back from trusted hash back to genesis.
2. Download blocks one at a time starting from genesis upwards.
    - The difference here would be that instead of executing the blocks, we would just download the global state trie from our peers.
3. When we're in the current protocol version and can't fetch any more blocks, stop.

After completing this process, the node will have the complete chain and all historical global state tries.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Going into more details of the process explained above, it would look like this:

1. Download the header corresponding to the trusted hash.
2. While the current header's height isn't 0, download the parent header by hash.
3. When height 0 is reached, read the genesis set of validators for validating finality signatures.
4. Download subsequent blocks with all the deploys, the global state trie and finality signatures in a loop.
5. When no next block is available, stop.

If we reach a block with a newer protocol version than our own during this process, we stop and attempt to upgrade.

## Drawbacks

[drawbacks]: #drawbacks

With this design, there is a larger risk of losing the history of the network, as the number of complete copies would be dramatically decreased. However, the risk will still be small, and the lowered costs of running a regular nodes seem to be worth it.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Archival nodes have to download _everything_ when syncing, so we need a process which satisfies that condition. The pre-fast-sync process achieved this goal, so an alternative could be to keep it and support it alongside fast sync, but:

- The process detailed above is essentially equivalent to the old syncing process.
- The old syncing process would have to be improved.
- Keeping the two alongside each other is more code to be maintained.

The first point means that there is no clear gain from keeping the old process, while there are clear drawbacks.

## Prior art

[prior-art]: #prior-art

None known, except our own old syncing process.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

Do archival nodes need to satisfy some other requirements, except being able to download the whole chain with all its data?

## Future possibilities

[future-possibilities]: #future-possibilities

None at the moment.
