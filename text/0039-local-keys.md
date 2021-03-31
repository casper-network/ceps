# Add local keys

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

This document describes the support for improved local keys, which intends to resolve past problems that lead to their removal. Contract developers will use this to store data in separate keyspace prefixed by the URef address provisioned to a user by the system.

## Motivation

[motivation]: #motivation

There were local keys that historically had a separate partition in the storage. The caller had to hash the data before writing it to fit the data in a 32-byte limit. When we introduced contract headers, versioning was a problem as, logically, local keys and contracts were on different storage layers. That meant two different contract versions could access data under the duplicate local keys.
Since the mint contract used the remaining local keys only to store balances, we decided to deprecate local keys and further remove them by introducing a separate "balance" keyspace. This further limits our contract platform's usage as implementing different token standards might be impossible or inefficient at best.

This proposal intends to fill in the functionality gap and resolves past problems which led to the removal of local keys.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Implementation of revised local keys will involve adding a separate keyspace to the storage layer. Each operation on the local key will require a "seed" URef, which the execution engine will authenticate.

Under the hood, local addresses have a length limit of 32 bytes, and the system will generate an address in the storage by hashing the URef address and key bytes together through API access.

Example usage from the perspective of a contract developer:

```rust
fn call() {
    // Initialize
    let local_uref: URef = storage::create_local();
    runtime::put_key("local_data", local_uref.into());

    // Write data
    storage::write_local(local_uref, b"key", "value".to_string());

    // Read data
    let value: String = storage::read_local(local_uref, b"key");
    assert_eq!(value, "value".to_string());

    // List stored data under URef
    let key_bytes = storage::list_local(local_uref);
    assert_eq!(
        keys,
        vec![
            b"key".to_vec(),
        ]
    );
}
```

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

No existing parts of the system will change due to the proposal's nature, which intends only to extend the system's affected parts.

This document intends to introduce new host functions to create and modify local keys. Implementation will also match the new functionality in contract API for developers.

This proposal introduces a hashing function under the hood due to the current trie implementation limitation that does not support keys longer than 32 bytes. As hashing is a lossy function, we still want to retain developers' ability to list data referenced through local key URef. The system will maintain a secondary index under separate space, consisting of all the keys for each (key, value) pair stored under the "local key.". Note that this will happen only internally, as once we can move to a prefix-based iteration in a larger space, there will be no secondary index involved. Therefore, the system will not expose access to the data under the index to developers besides the "list_local" API for future compatibility.

With this design decision, we will retain the possibility of adding a data migration in the future to read all (key, value) pairs through the index and move the data into an expanded space that will consist of a "URef" address and a hash of "key" bytes concatenated. We will programmatically replace the local keys' values with a pair that contains unchanged key bytes and a value itself during migration. The migration will remove a secondary index, and then we will rely on a prefix-based iteration mechanism in list_local.

### Key::Local

Adds a new variant that holds 32 bytes hashed value based on:

```
blake2b256(uref_addr || key_bytes)
```

Further operations such as read/write use the algorithm as mentioned earlier to access the `Key::Local` space.

### create_local

```rust
fn create_local() -> URef;
```

Host provisions an URef of "unit" type under the hood used internally to store a secondary index of keys. This function's caller should keep the URef mentioned above under account or contract's named keys, retaining his ability to add new data.

Newly created will have a type set to "unit" to hide the implementation details and discourage using "read" or "write" APIs to operate on local keyspace. The host will also secure this possibility on the backend, and inside of these host functions, a particular check will verify and forbid direct operations on `Key::Local` space.

### read_local

```rust
fn read_local(uref, key) -> T;
```

Reads data from a local key under "URef" and "key" bytes and returns a stored value. The system then verifies that URef is valid in the caller context and performs hashing of the "URef" address and the key bytes.

A secondary key index is not involved in this operation.

### write_local

```rust
fn write_local(uref, key, data);
```

Writes data under a key to a local key referenced by "URef.". The system then verifies that URef is in the execution context, and an additional check is in place to confirm if the "WRITE" permission bit is on.

If the given "key" is not present in the secondary index that "URef" points to, then it is extended with "key." bytes.

### add_local

```rust
fn add_local(uref, key, data);
```

Similar to `write_local,` but emits an `ADD` transformation for a given local key.

### list_local

```rust
fn list_local() -> Vec<Vec<u8>>;
```

Lists all key values under a URef.

## Drawbacks

[drawbacks]: #drawbacks

As explained above current limitation of key lengths inside a trie requires us to maintain a secondary index of keys under a URef to retain the ability to migrate into the unhashed, more extended keyspace for more efficient iteration.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Alternatives to using proposed local keys would be to store data as separate named keys. Doing this creates another problem - mainly bloat of account/contract's named keys which may increase WASM execution startup times as account (or contract) has to be deserialized before execution.

## Prior art

[prior-art]: #prior-art

In the past, we used to have `Key::Local,` which we first deprecated and then removed as we used it only in system contracts, and there was a better functionality in place to replace uses. This proposal builds on top of past implementation by fixing problems that it had along the way.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the CEP process before this gets merged?
- What related issues do you consider out of scope for this CEP that could be addressed in the future independently of the solution that comes out of this CEP?

## Future possibilities

[future-possibilities]: #future-possibilities

One unanswered question about local keys introduced by this CEP is listing all the "key"s contained within a URef from within runtime. Currently, to match existing functions such as "list_named_keys," this document proposes that runtime will deserialize all keys at once. In the future, however, it would be great if we'd introduce a way to load keys incrementally, possibly saving contract developers gas for use cases like getting only selected elements.

Once we land support for iteration over keys without loading everything into memory and a contract API to remove keys from the storage, contract developers can use this functionality to implement an efficient container that keeps keys and values under local storage. We can further consider landing such high-level concepts in the contract API as it would be a vast improvement over expensive reading and writing contents "BTreeMap." repeatedly.