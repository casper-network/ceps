# Extensible Block Headers and Bodies via Hashing Supporting Merkle Proofs

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0041](https://github.com/casperlabs/ceps/pull/0041)

This CEP proposes allowing for extensible block headers and bodies.  To achieve this, we propose changing the hashing procedure for these structures.  The new hashing system will support Merkle proofs. This will allow the secure transfer of partial data structures. Doing so will permit extending these structures in a backwards-compatible manner.

## Motivation

[motivation]: #motivation

There are three main reasons this will be useful:
1. The block header and body will be representable as content-addressed linked lists. It will be easy to add more fields to them should we ever need them.
2. This is important for Fast Sync to be able to securely download only the relevant parts of old blocks, increasing performance.
3. This will reduce the need for special handling of different versions of block headers and bodies.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Here we present the design for the block body. The design for the block header is similar, but complicated by having more fields.

The block body currently consists of three fields: the list of deploy hashes, the list of transfer hashes and the public key of the validator that proposed the block. The hash of the block body is calculated by just hashing the concatenated data. This means that in order to verify the correctness of a single field, we have to download the whole block body. Also, the block body is stored in a single hash → body database, which means that if we add another field to the body, old block bodies would become unreadable.

What we can do instead is hash the block headers and bodies in a way similar to generating Merkle trees - except in our case, the tree would be perfectly imbalanced (details to be found [below](#reference-level-explanation)).

By employing Merkle proofs, we will enable partial verification of block headers and bodies. For example, verifying the list of deploys for a block body will require downloading only that list and the hash of the rest of the body, significantly reducing the bandwidth usage. We will also be able to store parts of the body in a way that allows for both easy verification and easy extension of the block contents.

One key difference between blocks and block headers is storage.  The Fast Sync feature requires that storage of partial blocks be supported.  On the other hand, storage of partial headers is not required at this time.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Once again, we present the approach for block bodies for simplicity.

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
                                         hash(proposer)   SENTINEL
```

`SENTINEL` is a value that is very improbable to be a valid hash of some data - like, for example, the all-zeros hash (`000000...00`).

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

The change to the way of calculating the hash of the block header and body would be compatible with upgrades.

If we keep the ability of calculating hashes the old way, alongside the new way, and the software is aware of the protocol version at which the transition happened, it can always calculate the hash using the way that is correct for the given block. Thus, the new versions of the software would be compatible with the old blocks. On the other hand, older versions of the software should never have to have anything to do with the new blocks - if a node progressed past the upgrade at which the new block body format has been introduced, even if it starts an old version of the software, it would quickly realize that it's outdated, shut itself down and prompt the launcher to start a newer version.

### Hashing Primitives In Rust

Here we present the hashing primitives used for constructing the new hashes.

The first is a primitive for hashing pairs of hashes:

```rust
pub fn hash_pair(hash1: &Digest, hash2: &Digest) -> Digest {
    let mut to_hash = [0; Digest::LENGTH * 2];
    to_hash[..Digest::LENGTH].copy_from_slice(&(hash1.to_array())[..]);
    to_hash[Digest::LENGTH..].copy_from_slice(&(hash2.to_array())[..]);
    hash(&to_hash)
}
```

The second is a primitive for hashing slices. This follows the scheme for hashing a `BlockBody` described in the previous section. It is intended for hashing vectors where we care about order.  This is suited to hashing extendable data structures. It uses a [right fold][wikipedia_fold]. It may be implemented using the [`rfold`][rfold] method defined in the rust standard library.

```rust
pub const SENTINEL1: Digest = Digest([1; 32]);

pub fn hash_slice_rfold(slice: &[Digest]) -> Digest {
    slice
        .iter()
        .rfold(SENTINEL1, |prev, next| hash_pair(next, &prev))
}
```

Note we choose `SENTINEL0` for this primitive. Other primitives will use different sentinels. This is intended to help with debugging.

In this case `hash_slice_rfold(&[a, b, c])` effectively expands to:

```rust
hash_pair(a, &hash_pair(b, &hash_pair(c, &SENTINEL1)))
```

This scheme is suited to simple Merkle proofs and verification.  Hashing with a proof is almost identical to `hash_slice_rfold`.

```rust
fn hash_slice_with_proof(slice: &[Digest], proof: Digest) -> Digest {
    slice
        .iter()
        .rfold(proof, |prev, next| hash_pair(next, &prev))
}
```

The third hashing primitive constructs a [Merkle tree][wikipedia_merkle_tree]. It is intended for vectors where we do not care about order. The `deploy_hashes` field of `BlockBody` as previously discussed is an example.  When handed an empty list the procedure returns the sentinel `hash(&[])`.

```rust
pub const SENTINEL2: Digest = Digest([2; 32]);

fn hash_vec_merkle_tree(vec: Vec<Digest>) -> Digest {
    if vec.is_empty() {
        return SENTINEL2;
    };
    let mut vec = vec;
    let mut k = vec.len();
    while k != 1 {
        let j = k / 2 + k % 2;
        for i in 0..j {
            if (2 * i + 1) > k {
                vec[i] = vec[2 * i];
            } else {
                vec[i] = hash_pair(&vec[2 * i], &vec[2 * i + 1]);
            }
        }
        k = j;
    }
    vec[0]
}
```

The procedure is akin to [graph reduction][wikipedia_graph_reduction]. The procedure would hash `vec![a, b, c, d, e, f]` as follows:

```
a b c d e f
|/  |/  |/
g   h   i
| /   /
|/   /
j   k
| /
|/
l
```

Using the [`itertools`][itertools], the above may be written concisely with [`tree_fold1`][tree_fold1]:

```rust
fn hash_vec_merkle_tree(vec: Vec<Digest>) -> Digest {
    vec.into_iter()
        .tree_fold1(|x, y| hash_pair(&x, &y))
        .unwrap_or(SENTINEL2)
}
```

Based on `hash_vec_merkle_tree`, we can give a procedure for hashing `BTreeMap`s. This procedure uses the `ToBytes` trait to abstract away the keys and values of the map.

```rust
fn hash_btree_map<K, V>(btree_map: &BTreeMap<K, V>) -> Result<Digest, bytesrepr::Error>
where
    K: ToBytes,
    V: ToBytes,
{
    let mut kv_hashes: Vec<Digest> = Vec::with_capacity(btree_map.len());
    for (key, value) in btree_map.iter() {
        kv_hashes.push(hash_pair(&hash(key.to_bytes()?), &hash(value.to_bytes()?)))
    }
    Ok(hash_vec_merkle_tree(kv_hashes))
}
```

Hashing an option is similar.  Once again the sentinel hash `hash(&[])` is used to denote absence.

```rust
pub const SENTINEL0: Digest = Digest([0; 32]);

fn hash_option<T>(maybe_t: Option<T>) -> Result<Digest, bytesrepr::Error>
where
    T: ToBytes,
{
    match maybe_t {
        None => Ok(SENTINEL0),
        Some(t) => Ok(hash(t.to_bytes()?)),
    }
}
```

[wikipedia_fold]: https://en.wikipedia.org/wiki/Fold_(higher-order_function)#Linear_folds
[rfold]: https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html#method.rfold
[wikipedia_merkle_tree]: https://en.wikipedia.org/wiki/Merkle_tree
[wikipedia_graph_reduction]: https://en.wikipedia.org/wiki/Graph_reduction
[itertools]: https://docs.rs/itertools/latest/itertools/
[tree_fold1]:https://docs.rs/itertools/latest/itertools/trait.Itertools.html#method.tree_fold1

### Prototype Rust Data-Structure Hashing Methods

This section provides sketches of how hashing functions will be implemented for `BlockBody` and `BlockHeader`s.

The following is a prototype of how the new `hash()` method for `BlockBody` will look:

```rust
impl BlockBody {
    fn hash(&self) -> Digest {
        // Pattern match here leverages compiler to ensure every field is accounted for
        let BlockBody {
            deploy_hashes,
            transfer_hashes,
            proposer,
        } = self;

        let hashed_deploy_hashes =
            hash_vec_merkle_tree(deploy_hashes.iter().cloned().collect());
        let hashed_transfer_hashes =
            hash_vec_merkle_tree(transfer_hashes.iter().cloned().collect());
        let hashed_proposer = hash(&proposer.to_bytes());

        hash_slice_rfold(&[
            hashed_deploy_hashes,
            hashed_transfer_hashes,
            hashed_proposer,
        ])
    }
}
```

The hashing function for `BlockHeader` has more parts. Block headers include nested data structures. In particular, it includes information about the next era validators, as well as information necessary for calculating block rewards.

At the highest level, `BlockHeaders` are hashed as follows:

```rust
impl BlockHeader {
    pub fn hash(&self) -> BlockHash {
        let BlockHeader {
            parent_hash,
            era_id,
            body_hash,
            state_root_hash,
            era_end,
            height,
            timestamp,
            protocol_version,
            random_bit,
            accumulated_seed,
        } = self;

        let hashed_era_id = hash(era_id.to_bytes());
        let hashed_era_end = match era_end {
            None => SENTINEL,
            Some(era_end) => era_end.hash(),
        };
        let hashed_height = hash(height.to_bytes());
        let hashed_timestamp = hash(timestamp.to_bytes());
        let hashed_protocol_version = hash(protocol_version.to_bytes());
        let hashed_random_bit = hash(random_bit.to_bytes());

        hash_slice_rfold(&[
            parent_hash.0,
            hashed_era_id,
            *body_hash,
            *state_root_hash,
            hashed_era_end,
            hashed_height,
            hashed_timestamp,
            hashed_protocol_version,
            hashed_random_bit,
            *accumulated_seed,
        ])
        .into()
    }
}
```

Similarly `EraEnd` data-structures are hashed:

```rust
impl EraEnd {
    pub fn hash(&self) -> Digest {
        let EraEnd {
            next_era_validator_weights,
            era_report,
        } = self;
        let hashed_next_era_validator_weights = hash_btree_map(next_era_validator_weights);
        let hashed_era_report: Digest = era_report.hash();
        hash_slice_rfold(&[hashed_next_era_validator_weights, hashed_era_report])
    }
}
```

Finally, `EraReport`s are hashed:

```rust
impl EraReport {
    pub fn hash(&self) -> Digest
    {
        // Helper function to hash slice of validators
        fn hash_slice_of_validators(slice_of_validators: &[PublicKey]) -> Digest
        {
            let mut hashes = Vec::with_capacity(slice_of_validators.len());
            for validator in slice_of_validators {
                hashes.push(hash::hash(validator.to_bytes()))
            }
            hash::hash_vec_merkle_tree(hashes)
        }

        let EraReport {
            equivocators,
            inactive_validators,
            rewards,
        } = self;

        let hashed_equivocators = hash_slice_of_validators(equivocators);
        let hashed_inactive_validators = hash_slice_of_validators(inactive_validators);
        let hashed_rewards = hash::hash_btree_map(rewards);

        hash::hash_slice_rfold(&[
            hashed_equivocators,
            hashed_rewards,
            hashed_inactive_validators,
        ])
    }
}
```

## Drawbacks

[drawbacks]: #drawbacks

This solution introduces additional complexity in the storage components and leaks some implementation details of the `BlockBody` struct to it.

Also, it requires us to hardcode the protocol version at which we transition from the old block format to the new one, and keep old code alongside new, so that it can handle both properly.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

An alternative allowing us to expand the block format would be to just perform a database migration every time we change it (which should be rare enough). However, this doesn't answer the need for secure downloading of only parts of the block body. Merkle proofs are the industry standard for solving problems of this kind.

The CEP also proposes to keep the old blocks in the old format. An alternative would be to migrate the old database of the block bodies to the new format. However, since block body hashes are immutable, the keys after the migration wouldn't always be the hash of the value stored under that key, which would make checking the integrity of storage much harder, if not impossible.

Another alternative would be not to store partial block bodies at all. For the purposes of preventing replay attacks, we only need to know which deploys were included in past blocks. This information could be stored in the form of a `deploy_hash → block_hash` database. We currently have such a map as an index (so this data is lost on node restart and requires the block body data to be repopulated), and a `deploy_metadata` database that stores the block hash along with execution results - the drawback of the latter is that it only stores entries for _executed_ deploys, and fast sync wouldn't execute past deploys, so this database wouldn't be populated.

## Prior art

[prior-art]: #prior-art

Merkle proofs are widely used in applications that require verification of partial data, eg. Bitcoin or BitTorrent.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

None at the moment.

## Future possibilities

[future-possibilities]: #future-possibilities

Future possibilities include adding more fields to the block body. In fact, the purpose of this CEP is to unlock such a possibility.
