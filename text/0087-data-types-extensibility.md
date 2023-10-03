# Data types extensibility

## Summary

[summary]: #summary

Backward compatible extensibility of fundamental data types used in `casper-node`, such as for example `Block` or `Deploy` is crucial from the perspective of efficient development of new features. Currently, any change done to the data structure is breaking. The extensibility mechanism we allow to extend the types while maintaining backwards compatibility.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

This feature will bring additional compatibility layer in a form of an `enum` wrapper that ties the current (aka legacy) version of the data type and all the updated versions, like so:
```rust
pub enum BlockBody {
    V1(BlockBodyV1),
    V2(BlockBodyV2),
    V3(BlockBodyV3),
    ...
}
```

There is no migration of existing data. If a body is stored in the DB as `BlockBodyV1` it'll stay intact and the components that operate on DB will be modified to understand all variants.

It is allowed to used the specific, most recent version directly in the code, because the idea is that only single (most recent) version of a particular type is considered "current". Only parts of the system that need to interact with legacy versions should use the `enum` wrapper, for example, the block synchronizer which must be able to work with historical blocks. This approach will not be enforced, therefore it is possible for all components to work with the `enum` wrapper and react as necessary to handle different variants of data.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

In technical detail, this proposal introduces a new `enum` wrapper type for every data struct we need to enable the extensibility for. For example, this is how the new `Block` type might be represented:

```rust
pub enum Block {
    V1(BlockV1),
    V2(BlockV2),
}

pub struct BlockV1 {
    pub(super) hash: BlockHash,
    pub(super) header: BlockHeader,
    pub(super) body: BlockBodyV1,
}

pub struct BlockV2 {
    pub(super) hash: BlockHash,
    pub(super) header: BlockHeader,
    pub(super) body: BlockBodyV2,
    pub(super) new_member: bool,
```

Each specific variant is (de)serializable and there is also a (de)serialization implemented for the top level `enum` wrapper which uses the predefined `TAG`s, for example:

```rust
    fn to_bytes(&self) -> Result<Vec<u8>, bytesrepr::Error> {
        let mut buffer = bytesrepr::allocate_buffer(self)?;
        match self {
            Block::V1(v1) => {
                buffer.insert(0, BLOCK_V1_TAG);
                buffer.extend(v1.to_bytes()?);
            }
            Block::V2(v2) => {
                buffer.insert(0, BLOCK_V2_TAG);
                buffer.extend(v2.to_bytes()?);
            }
        }
        Ok(buffer)
    }
```

Components that never need to touch the legacy types should directly use the most recent version of the datatype. For example, going forward, we never need to gossip "legacy" blocks during the normal chain operation, therefore the `Gossiper` should work directly with `BlockV2`.

The `Fetcher` on the other hand must be able to deal with all variants of the datatype, because it must be able to fetch and understand all historical blocks up to genesis.

### Storage

Storage is the main component that should understand and work with the versioned types. For each data type that needs to be extensible we create a separate DB which will be used to store the top-level `enum` wrapper. The legacy DB is left intact and it's going to be accessed only when old-enough data is requested. Therefore, upon the request to the storage being sent, storage will first look for a particular item in the new database and only if it is not found it'll reach out to the legacy DB and try to obtain data from there.

This is a pseudocode of how a block body retrieval function could be written:
```rust
fn get_body_for_block_header(hash, db_current, db_legacy)
) -> Option<BlockBody> {
    let body = get_value(hash, db_current);
    if body.is_none() {
        get_value(hash, db_legacy)
    }
    else {
        body
    }
}
```
There is no noticeable performance degradation expected because most of the requests will be saturated from the new DB and we'll fall back to the legacy one only for limited use cases (like historical sync, which in most cases is a one-time operation).

Upon storage initialization, indices are built based on the content of both DBs. Similarly, all utility functions, like `value_exists()` will be updated to check both databases.

## Alternatives

[alternatives]: #alternatives
### Magic bytes

Upon serialization we add the magic-string prefix to the serialized bytes. Similarly, upon deserialization we first check for existence of such magic-string and decide whether we're gonna be deserializing new or legacy data type.

Such approach has already been used (`UNBONDING_PURSE_V2_MAGIC_BYTES`) but it was rejected as a general solution mainly due to these reasons:
1. While the probability is low, magic strings are prone to collisions, especially when attached to types for which we cannot reason about what bytes are at the beginning of a struct. For example, `Block` starts with `BlockHash` which can be any combination of bytes. This approach would work better for types which starts, for example, with a `Vec` or `String` because we can reason that in such case first 4 bytes would be rather small as they describe the size of the collection.
2. To minimize the collision risk the magic-string should consist of at least several bytes. This may induce a significant data overhead if the type we're going to extend is small, such as `struct EraId(u64)`.

### Key-value map

Serialization code converts every struct to a `BTreeMap` collection and such collection is then serialized. For example, the `Block` serialization pseudocode may look like this:
```rust
fn to_bytes(&self) -> Result<Vec<u8>, bytesrepr::Error> {
        let mut field_map = BTreeMap::<String, Bytes>::new();
        field_map.insert("hash".to_string(), Bytes::from(self.hash.to_bytes()?));
        field_map.insert("header".to_string(), Bytes::from(self.header.to_bytes()?));
        field_map.insert("body".to_string(), Bytes::from(self.body.to_bytes()?));
        let bytes = field_map.to_bytes()?;
        ret.extend(bytes);
        Ok(ret)
    }
```
It's worth noting that this alone doesn't solve the extensibility. We'd still need some kind of magic-string to check if we're about to deserialize new or legacy type. It'll however, make the extensibility future-proof, because once the type is "migrated" to the `field_map`, we can keep adding as many new fields as we like without affecting (de)serialization compatibility.

### Self-describing format

Similar to the above, but instead of converting the struct to `field_map` we'll prepend the standardized header which will describe the structure of the datatype. Alternatively, we may consider adding the explicit version number (`u8`) to each type.

### `Versionize` crate

Use the `versionize` crate, which gives the versioning capability out-of-the box, but is strongly tied to the `bincode` backend. This possibility was not thoroughly explored.