# Add dictionary keys

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0039](https://github.com/casperlabs/ceps/pull/0039)

This document describes the support for dictionary keys, which intends to supersede previous `local keys` and resolve problems that lead to its removal. Contract developers will use this to store data in separate keyspace prefixed by the URef address provisioned to a user by the system.

## Motivation

[motivation]: #motivation

There were local keys that historically had a separate partition in the storage. The caller had to hash the data before writing it to fit the data in a 32-byte limit. When we introduced contract headers, versioning was a problem as, logically, local keys and contracts were on different storage layers. That meant two different contract versions could access data under the duplicate local keys.
Since the mint contract used the remaining local keys only to store balances, we decided to deprecate local keys and remove them by introducing a separate "balance" keyspace. This further limits our contract platform's usage as implementing different token standards might be impossible or inefficient at best.

This proposal intends to fill in the functionality gap and resolve past problems of local keys under a new name: a dictionary API.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Implementation of a dictionary will involve adding a separate keyspace to the storage layer. Each operation on the dictionary key will require a "seed" URef, which the execution engine will authenticate.

Under the hood, dictionary addresses have a length limit of 32 bytes, and the system will generate an address in the storage by hashing the URef address and key bytes together through API access.

Example usage from the perspective of a contract developer:

```rust
fn call() {
    // Initialize and store the URef in named keys
    let dictionary_uref: URef = storage::new_dictionary("data");

    // Write data
    storage::dictionary_put(dictionary_uref, b"key", "value".to_string());

    // Read data
    let value: String = storage::dictionary_get(dictionary_uref, b"key");
    assert_eq!(value, "value".to_string());
}
```

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

No existing system parts will change due to the proposal's nature, which intends only to extend the system's affected parts.

This document intends to introduce new host functions to create and modify dictionary keys. Implementation will also match the new functionality in contract API for developers.

This proposal introduces a hashing function under the hood due to the current trie implementation limitation that does not support keys longer than 32 bytes. As hashing is a lossy function, we still want to retain developers' ability to list data referenced through dictionary URef. The system will transparently wrap written data in a structure which will preserve the original `URef` and actual key bytes. This operation is completely hidden from the user and considered an implementation detail.

Whenever we are ready to migrate the data from short 32-byte space into 64-byte, we will employ an expensive one-time sweep over old Dictionary keys and rehash them into new, more extended space.

### Key::Dictionary

Adds a new variant that holds 32 bytes hashed value based on:

```
blake2b256(uref_addr || key_bytes)
```

Further operations such as reading or writing use the earlier algorithm to access the `Key::Dictionary` space.

### new_dictionary

```rust
fn new_dictionary(name: &str) -> URef;
```

Host provisions an URef of "unit" type under the hood used internally to secure access to the dictionary. Resulting URef will be automatically put into named keys, retaining the ability to add new data.

The name argument should not be empty, and additionally, the caller should not have a given key set already.

Newly created URef will have a type set to "unit" to hide the implementation details and discourage using "read" or "write" APIs to operate on dictionary keyspace. The host will also secure this possibility on the backend, and inside of these host functions, a particular check will verify and forbid direct operations on the `Key::Dictionary` space.

### dictionary_get

```rust
fn dictionary_get(dictionary_uref, key) -> T;
```

Reads data from a dictionary under "URef" and "key" bytes and returns a stored value. The system then verifies that URef is valid in the caller context, has a "READ" permission bit, and performs hashing of the "URef" address and the key bytes.

A secondary key index is not involved in this operation.

### dictionary_put

```rust
fn dictionary_put(dictionary_uref, key, data);
```

Writes data under a key to a dictionary referenced by "URef.". The system then verifies that URef is in the execution context, and an additional check is in place to confirm if the "WRITE" permission bit is on.



## Drawbacks

[drawbacks]: #drawbacks

As explained above current limitation of key lengths inside a trie requires us to write an original value together with the original dictionary URef, and actual key bytes to retain the ability to migrate into the unhashed, more extended keyspace for more efficient iteration. However, no performance loss will happen as part of this operation as this operation only happens through reads (incl. querying) and writes.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Alternatives to using proposed dictionary keys would be to store data as separate named keys. However, this creates another problem - mainly bloat of account/contract's named keys which may increase WASM execution times as account (or contract) is always deserialized before execution. Not to mention continuous overwriting of global state keys, which also may bloat storage space.

## Prior art

[prior-art]: #prior-art

In the past, we used to have `Key::Local,` which we first deprecated and then removed as we used it only in system contracts, and there was a better functionality in place to replace uses. This proposal builds on top of past implementation by fixing problems that it had along the way under a new name that better reflects its functionality.

## Future possibilities

[future-possibilities]: #future-possibilities

Once we land support for iteration over keys without loading everything into memory and a contract API to remove keys from the storage, contract developers can use this functionality to implement an efficient container that keeps keys and values under dictionary. We can further consider landing such high-level concepts in the contract API as it would be a vast improvement over expensive reading and writing contents "BTreeMap." repeatedly.
