# Extensible Block Body

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

Modify the internal structure of the block body, so that it will be easy to add new fields to it in the future in a backwards-compatible manner.

## Motivation

[motivation]: #motivation

There are two main reasons this will be useful:
1. As the block body will essentially become a content-addressed linked list, it will get really easy to add more fields to it, should we ever need them.
2. This is important for Fast Sync to be able to securely download only the relevant parts of old blocks, increasing performance.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The block body currently consists of three fields: the list of deploy hashes, the list of transfer hashes and the public key of the validator that proposed the block. The hash of the block body is calculated by just hashing the concatenated data. This means that in order to verify the correctness of a single field, we have to download the whole block body. Also, the block body is stored in a single hash → body database, which means that if we add another field to the body, old block bodies would become unreadable.

What we can do instead is hash the block bodies in a way similar to generating Merkle trees - except in our case, the tree would be perfectly imbalanced (details to be found [below](#reference-level-explanation)).

By employing Merkle proofs, we will enable partial verification of block bodies. For example, verifying the list of deploys will require downloading only that list and the hash of the rest of the body, significantly reducing the bandwidth usage. We will also be able to store parts of the body in a way that allows for both easy verification and easy extension of the block contents.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The hash of the block body would be redefined as the following:

- hash of the body is the hash of `[hash of the deploy hashes list, hash_2]`
- `hash_2` is the hash of `[hash of the transfer hashes list, hash_3]`
- `hash_3` is the hash of `[hash of the proposer, hash_4]`
- `hash_4` is the hash of an empty bytestring.

This can be illustrated by the following diagram:

```
                   body_hash
                   /      \
 hash(deploy_hashes)      hash_2____________
                         /                  \
                   hash(transfer_hashes)     hash_3_______
                                            /             \
                                         hash(proposer)   hash("")
```

So, for example, to securely download only the transfer hashes stored in a block with a known body hash, we will additionaly require only the hash of the deploy hashes list and `hash_3`. This will allow us to calculate `hash_2` and `body_hash`, and to verify that the body hash matches the one we have.

The block bodies would be stored the following way:
- A `block_body_merkle` database would store a key-value map of `body_hash → vec![hash(deploy_hashes), hash(transfer_hashes), hash(proposer)]`. The type `Vec<Digest>` of the value will allow for storing blocks with more fields in the same database without the risk of causing problems with deserialization.
- A `deploy_hashes` database would store the lists of deploy hashes keyed by `hash(deploy_hashes)`.
- A `transfer_hashes` database would store the lists of transfer hashes keyed by `hash(transfer_hashes)`.
- A `proposer` database would store the proposers keyed by `hash(proposer)`.

The new way of hashing and new databases would start being used in a specific, chosen protocol version. The blocks in this protocol version and above would have their bodies hashed using the new way and be stored in the new databases. Older blocks would be kept in the old database.

This is to ensure that we can still verify the integrity of storage after we transition to the new format. Changing the body hash of the old blocks would also change the hash of the block and hashes of all the later blocks, so it is not an option. So in order to be able to verify that the stored data still corresponds to the hashes from the blocks, we need to be able to calculate them the old way even after we start using the new way for new blocks.

The code reading the blocks from storage would then have to be aware of the protocol version at which the transition happened, in order to read the block bodies from the correct databases regardless of whether it's a new or an old block.

### The new block body hash and upgrades

The change to the way of calculating the hash of the block body would be compatible with upgrades.

If we keep the ability of calculating hashes the old way, alongside the new way, and the software is aware of the protocol version at which the transition happened, it can always calculate the hash using the way that is correct for the given block. Thus, the new versions of the software would be compatible with the old blocks. On the other hand, older versions of the software should never have to have anything to do with the new blocks - if a node progressed past the upgrade at which the new block body format has been introduced, even if it starts an old version of the software, it would quickly realize that it's outdated, shut itself down and prompt the launcher to start a newer version.

## Drawbacks

[drawbacks]: #drawbacks

This solution introduces additional complexity in the storage components and leaks some implementation details of the `BlockBody` struct to it.

Also, it requires us to hardcode the protocol version at which we transition from the old block format to the new one, and keep old code alongside new, so that it can handle both properly.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

An alternative allowing us to expand the block format would be to just perform a database migration every time we change it (which should be rare enough). However, this doesn't answer the need for secure downloading of only parts of the block body. Merkle proofs are the industry standard for solving problems of this kind.

The CEP also proposes to keep the old blocks in the old format. An alternative would be to migrate the old database of the block bodies to the new format. However, since block body hashes are immutable, the keys after the migration wouldn't always be the hash of the value stored under that key, which would make checking the integrity of storage much harder, if not impossible.

## Prior art

[prior-art]: #prior-art

Merkle proofs are widely used in applications that require verification of partial data, eg. Bitcoin or BitTorrent.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

None at the moment.

## Future possibilities

[future-possibilities]: #future-possibilities

Future possibilities include adding more fields to the block body. In fact, the purpose of this CEP is to unlock such a possibility.
