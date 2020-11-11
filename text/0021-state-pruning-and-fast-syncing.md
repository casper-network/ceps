# State pruning and fast synching

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0021)

Reduce the time required to sync and the disk usage of nodes by allowing both transfer and offloading of state (global state as well storage).

## Motivation

[motivation]: #motivation

Disk usage of a node goes up continously with the current implementation, increasing the required resources to host a node, potentially posing a significant barrier to entry for validators. Not all of this state is necessary for operation, some is only ever retrieved due to the current implementation of the joining process, which requires all historical data since genesis to derive an up-to-date global state.

It should be possible to reduce this overhead for nodes that are not participating in a record-keeping function (colloqially called "archival nodes"), bringing down disk space and CPU time, as well as reducing strain on the network as a whole.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

We are addressing two core business concerns with this CEP, namely

* **state-pruning**: removing data from disk that is no longer required for the current operation of the node participating as "just" a validator, and
* **fast-syncing**: speeding up the process by which a node joins the network, making it faster than the current approach of replaying every block starting at genesis.

Considering **state** we partition three broad classes of state, *trie-store*, *storage* and *consensus* state. The trie-store holds the so-called *global state*, which is the state as seen by all smart contracts running on the blockchain. The storage state is handled by the storage component and deals mainly with blocks, deploys and related data such as signatures. State inside consensus component is a set of directed-acyclic graphs holding the information about the running consensus process.

This CEP focus solely on the first two classes of state (trie-store and global state). While consensus state could benefit from a fast-syncing process, it is currently not persisted to disk and evicted periodically, for this reason we defer extending this proposal to its data structures future work for another CEP.

### Trie-store pruning

To be able to prune state (and thus reclaim disk space), we first need to identify which state can be discarded safely without impeding operation. For the trie store this data is rather small under normal circumstances:

Any insertion of data grows the global state irrevocably and this new data cannot be trimmed. If data is inserted in a leaf node, a new branch leading up to root hash is added, invalidating only a single branch, which may already be compressed. The total prunable amount of data is thus expected to be rather small for a single insert. Deletes are currently not implemented, but offer state savings similar in size.

The most promising operation for recovering space via pruning is the replacement of a leaf node of substantial size, which will free up the memory used by the previous incarnation of said leaf. Unfortunately the largest leaf nodes in the trie store are typically WASM blobs, thus state pruning would offer the largest size savings if contracts were frequently replaced by newer versions, which is not likely (or at least not desirable) to happen.

The current implementation of the trie store has a small wrinkle in that when executing a set of operations like applying all deploy operations of a block, it creates and stores intermediate root hashes that will soon be unreachable. This adds to the set of nodes that can be pruned. If this detail were fixed, unless there is an increase in churn of WASM code, the savings from trie-store pruning are potentially not substantial.

These considerations aside, pruning for the trie store can be implemented using a simple `O(n)` (assuming a hash set) garbage collection algorithm:

1. Decide on a set of root hashes to keep.
1. Traverse the tree of each root hash, adding each to a set of "nodes to keep".
1. Iterate over the backing store, deleting all keys not in the "nodes to keep" set.

A potentially more invasive mark-and-sweep version of this algorithm allows for online-collection by adding a `keep` flag to each node that defaults to `true` for every value inserted into the backing store and using the following algorithm:

1. Set all `keep` flags to `false` for all currently known keys.
1. Decide on a set of root hashes to keep.
1. Traverse the tree of each root hash, setting `keep` to `true` for every visited key.
1. Iterate over the backing store, deleting all keys not marked `keep`.

This algorithm has the advantage of using less memory and being suspendable.

This leaves us with the problem of deciding which root hashes to keep, which will be addressed later.

### Storage pruning

Pruning of the storage offers more substantial savings than the trie store, as most disk space is to consumed by deploys. Each block has associated data (deploys, finality signatures, etc.) that is unlikely to be shared by other blocks, as deploys are only added to one block and finality signatures are also unique. Evicting blocks from the store should thus free up space proportionally on average.

By deciding to "cut" the block chain at a specific parent, we can apply a similar garbage collection algorithm:

1. Decide which blocks to keep and add them to the "objects to keep" set (*i.e.* a starting block and all descendents).
1. For each block, visit all of its dependencies (deploys, finality signatures) and add them to an "objects to keep" set.
1. Visit each block, deploy and finality signature, remove them if not inside the "objects to keep" set.

In a manner similar to the trie store, we can employ a mark-and-sweep version of this pruning algorithm as well.

This again reduces the problem on selecting which blocks to keep.

### Pruning summary

Both trie-store as well as storage can be trimmed down by garbage collection, with savings in the latter probably being more substantial than the former.

In both cases the problem can reduced to identifying which root hashes or blocks must be kept and which can be discarded. Once this information is available, garbage collection becomes straight forward.

An "archival node" is a node on which garbage collection is either disabled or restricted to non-transitive state information, *i.e.* it keeps all root hashes except those created in the midst of execution.

### Fast-syncing

Fast syncing, while beneficial in its own right, is also of interest from a storage perspective, as being able to recover information quickly from the network or external sources influences our decisions about what information must be kept.

The "fast" part of fast-syncing, as compared to the status quo, is realized from two factors, namely

* **side-loading** data from outside the network through external sources and
* **transfering** state, namely trie-store state, which is currently not transferrable at all.

### Side-loading

We create a common trait for Blocks, Deploys or any other transferable object that states how it can be serialized and defines a hash function for its serialized representation, as well as a key to separate the hash space. As an example, a block whose hash is `5A4A55A8CA87B584960DCB0673DC1260292991EC6D2B8F214701B5224FB141D2` would be adressable as `blockv1_5A4A55A8CA87B584960DCB0673DC1260292991EC6D2B8F214701B5224FB141D2`. The `blockv1` denotes a complete scheme, that is it also pins the serialization format. While it is not necessary to separate the hash value space to avoid collisions, the prefix allows for multiple representations, should the format of a serialization or an internal data structure change at some point. We refer to data structures that satisfy this condition as *objects* from here on out.

**Note**: The defunct CEP3 proposed hashing the serialized representation of an object instead of the object itself, which makes it a lot easier for external stores to handle these, as integrity can be checked without deserializing first. However, hashing the internal object minimizes the impact on the existing codebase instead.

We can then define a generic mechanism for requesting these objects from other nodes. The true value however is in being able to serialize these to static external storage.

### S3, HTTP and read/write, read-only stores

As objects are unchanging, they can be readily be side-loaded from an HTTP-server that acts as a read-only store. As an example, should a node need to obtain the block `5A4A55A8CA87B584960DCB0673DC1260292991EC6D2B8F214701B5224FB141D2`, it can follow the following steps:

1. Check one or more configured external HTTP stores by calling `GET store/objects/5A4A55A8CA87B584960DCB0673DC1260292991EC6D2B8F214701B5224FB141D2`
1. If the request is a hit, try to download the object, deserialize it and validate the hash. Should any of these operations fail, discard and continue. Otherwise return.
1. Ask the network (typically a node is known that is required to have it) for the object in the same manner.

In the same manner, an S3-compatible API can be employed. Since many S3 providers allow making stored objects public, it would be highly recommendable to use a similar structure for the keys, as a result, every object uploaded to S3 automatically becomes read-only fetchable by other nodes that have access to the S3 bucket in real-time.

Nodes configured to keep the external store up-to-date will, upon receiving a new object, attempt to upload it to the store, by first checking if it exists already and uploading it if it does not.

With these measures in place almost all currently persisted state can be backed up to a service provider. It is up to operators or the Casper organisation to maintain these stores, which can be hosted on decentralized services like [storaj](https://storj.io/), providers like Amazon, Google, or self-hosted, possibly private caches like [minio](https://min.io/).

### Transfering state from the trie store

Objects like blocks, deploys and similar are trivially convertable into objects. For the trie store it should also be possible, as every vertex on the tree is either a leaf, a node or an extension. The hash matches what the store calls a pointer, thus a recovery of a root hash `4813494D137E1631BBA301D5ACAB6E7BB7AA74CE1185D456565EF51D737677B2` from an external store or the network would look like this:

0. Initialize the retrieval queue with `4813494D137E1631BBA301D5ACAB6E7BB7AA74CE1185D456565EF51D737677B2`.
0. For every object in the retrieval queue, remove those already in the trie store. If the queue is now empty, exit.
0. Request all objects though the trie scheme, e.g. `triev1_4813494D137E1631BBA301D5ACAB6E7BB7AA74CE1185D456565EF51D737677B2`, which should deserialize to a `Trie::Node`.
0. After successful deserialization, add all descendants of retrieved object into the retrieval queue, or abort with an error.
0. Go to 1.

### Optimization of retrieval

Retrieval of any object can be optimized by batching, which will not be discussed in depth here. The general idea is to request multiple objects in a single request, and/or provide pre-assembled pack files that group data that is often requested at the same time, like a block and its deploys or the children of a subtree of a global state hash.

This is an optimization that should be deferred for now though.

### Putting everything together

With a combination of the proposals above, a node's joining process from a trusted hash ideally becomes this:

1. Download the block given in the trusted hash, sped-up through external caches.
2. Download the global state as described earlier directly, without the need for executing previous blocks.
3. The node can now join the network by following the remainder of the existing joining process.

To save space, the node is safe to remove at least all blocks that are older than the current block minus the unbonding period. It thus fills the following two sets:

* blocks-to-keep: All blocks that are younger than `now() - unbonding period`.
* root-hashes-to-keep: All root hashes that are referenced by any block of blocks-to-keep.

The node can now trigger the garbage collection process (and do so periodically if desired) to evict old data.

It is important that the code is structured in a way that an absence of any old data does not trigger an error, but a fetch of said data from an external source or the network. In general it is advisable to structure components this way, but this extends the scope of this CEP.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

There is no reference level explanation at this time, except for a proposal for the object serialization trait. This particular version requires no changes to the existing objects' structures.

**NOTE: THE CODE BELOW IS A SKETCH TO ILLUSTRATE THE CEP ABOVE AND IN NO SHAPE OR FORM FINAL**

```rust
/// A scheme for object serialization.
trait Scheme {
    /// The prefix the scheme uses to distinguish itself.
    const PREFIX: &'static str;

    /// The hash function used to hash `Object`s.
    type Digest : ToString;

    /// The actual object handled by the `Scheme`.
    type Object;

    /// Deserialize a serialized object.
    fn deserialize(data: &[u8]) -> Self::Object;

    /// Return the key used to pass a serialized object around.
    fn name(digest: Digest) -> String {
        format!("{}_{}", Self::PREFIX, digest.to_string())
    }

    /// Serialize an object instance.
    ///
    /// Does not use genericized traits like `Serialize`, as the concrete
    /// serialization method may differ between versions or schemes.
    fn serialize(object: &Self::Object, output: &mut io::Write) -> io::Result<()>;
}

/// Block object scheme, version 1.
struct BlockV1;

impl Scheme for BlockV1 {
    const PREFIX: &'static str = "blockv1";
    type Digest = BlockHash;
    type Object = Block;

    fn deserialize(data: &[u8]) -> Self::Object {
        // ... deserialize the block using `FromBytes`
    }

    fn serialize(object: &Self::Object, output: &mut io::Write) -> io::Result<()> {
        // ... serializes the block into `output` using `ToBytes`
    }
}

// Example: Request the retrieval of an object using a specific scheme.
effect_builder.retrieve_object::<BlockV1>::(my_object_hash)

// TODO: Consider alternative approaches that result in object-safe schemes (hard).
```

## Drawbacks

[drawbacks]: #drawbacks

Other than implementation effort, I can not think of any drawbacks at the time of this writing.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Previously an alternative for sharing global state according to the following-scheme was brought up:

* Divide the tree below a trie store root hash into long strips, one per level.
* Divide the resulting strips into chunks of 1 MB in size, allowing them to be sent one-by-one to peers.

In general, this approach has the advantage of being guaranteed an efficient chunking of the state, as small vertices will automatically be grouped together and large ones chunked, thus keeping overhead very reasonable. In contrast, the scheme proposed above does not exhibit this property and likely requires packing or batching to be competitive with this approach.

The approach is still not preferred as it has two crucial drawbacks: It does not naturally take advantage of the deduplicated nature of the trie store, even slightly differing global states being transferred would likely require an almost complete transfer. Additionally it does not offer easy access to the data "at-rest", i.e. when in a static store like the HTTP or S3 caches proposed above.

## Prior art

[prior-art]: #prior-art

- [CEP3](https://github.com/CasperLabs/ceps/pull/3)

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- A concrete strategy on how to move forward, piece-by-piece as follow-up work.

## Future possibilities

[future-possibilities]: #future-possibilities

* Should consensus state ever be persisted using the storage component, it could be shared in the same manner. Similarly it could be transferred in the same vein.

* Adopting an essentially "stateless" model for many components by simplifying on-demand retrieval could obsolete the need for a separate joining process.
