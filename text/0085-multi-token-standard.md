# Casper Multi-Token Standard 
CEP PR: casperlabs/ceps#0085

## Summary
A standard that enables multiple fungible and non-fungible tokens to exist within a single smart contract instance on a Casper network.

## Motivation
We aim to provide a reference for the Multi Token Standard that in design and quality that should be adhered to for optimal execution of contracts upon the Casper blockchain system.

A Multi-Token standard enables functionality that cannot be served by Fungible and Non-Fungible token standards separately. Creating a smart contract that combines these two features brings Casper in line with other top-tier blockchain platforms, and bolsters the use cases available on Casper networks. This standard ostensibly allows for limited-supply fungible tokens with added information, i.e., digital admission tickets, open-edition art and so on.

## Guide-level Explanation

### Interface

Any compliant contract should contain the following endpoints:

#### balance_of

Returns the amount of a specified token that a given address holds.

```rust
fn balance_of(account: Key, id: U256) -> U256
```

#### balance_of_batch

Returns the balance of a batch of accounts for a specified token.

```rust
fn balance_of_batch(accounts: Vec<Key>, ids: Vec<U256) -> Vec<U256>
```

#### mint

Mints an amount of the specified token and places it in the balance of the `recipient`.

```rust
fn mint(recipient: Key, id: U256, amount: U256)
```

#### batch_mint

Batch mints the provided amount of multiple tokens and places them in the balance of the `recipient`.

```rust
fn batch_mint(recipient: Key, ids: Vec<U256>, amounts: Vec<U256>)
```

#### burn

Eliminates the provided amount of a specified token from the balance of the owner and the overall total supply.

```rust
fn burn(owner: Key, id: U256, amount: U256)
```

#### batch_burn

Burns a batch of multiple tokens from the specified owner.

```rust
fn batch_burn(owner: Key, ids: Vec<U256>, amounts: Vec<U256>)
```

#### set_approval_for_all

Approves an account to act as an operator for the calling account, allowing the new account to take actions with the owner's tokens.

```rust
fn set_approval_for_all(operator: Key, approved: Boolean)
```

#### is_approved_for_all

Returns a `boolean` response indicating whether the operator is approved to act on behalf of an owner.

```rust
fn is_approved_for_all(owner: Key, operator: Key) -> bool
```

#### safe_transfer_from

Transfers an amount of the specified tokens from the `sender` to the `recipient`.

```rust
fn safe_transfer_from(from: Key, to: Key, id: U256, amount: U256)
```

#### safe_batch_transfer_from

Batch transfers specified amount of multiple tokens from the `sender` to the `recipient`.

```rust
fn safe_batch_transfer_from(ids: Vec<U256>, amounts: Vec<u256>, from: Key, to: Key)
```

#### supply_of

Returns the supply of the specified token.

```rust
fn supply_of(id: U256) -> U256
```

#### supply_of_batch

Returns the current supply for the specified tokens.

```rust
fn supply_of_batch(ids: Vec<U256>) -> Vec<U256>
```

#### total_supply_of

Returns the total supply of the specified token.

```rust
fn total_supply_of(id: U256) -> U256
```

#### total_supply_of_batch

Returns the total supply of the specified tokens.

```rust
fn total_supply_of_batch(ids: Vec<U256>) -> Vec<U256>
```

#### set_total_supply_of

Sets the total supply of the specified token.

```rust
fn set_total_supply_of(id: U256, total_supply: U256)
```

#### set_total_supply_of_batch

Sets the total supply of the batch of specified tokens.

```rust
fn set_total_supply_of_batch(ids: Vec<U256>, total_supplies: Vec<U256>)
```

#### uri

Returns a string URI for any off-chain resource associated with the token.

```rust
fn uri(id: Option<U256>) -> String
```

#### set_uri

Sets the assigned string as the token's URI.

```rust
fn set_uri(id: Option<u256>, uri: String)
```

#### is_non_fungible

Returns a `true` or `false` value for whether the specified token is non-fungible.

```rust
fn is_non_fungible(id: U256) -> bool
```

#### total_fungible_supply

Calculates the difference between the total supply and the current supply of a token. If the token is non-fungible, or if total supply has been reached, this returns 0.

```rust
fn total_fungible_supply(id: U256) -> U256
```

#### change_security

This is an admin endpoint to manipulate the security access granted to users. Each user can only possess one access group badge (None, Admin or Minter).

```rust
fn change_security(admin_list: Option<Vec<Key>>, minter_list: Option<Vec<Key>>, meta_list: Option<Vec<Key>>, none_list: Option<Vec<Key>>)
```

## Reference-level Explanation

### Batch Actions

This standard features the option to take actions on 'batches' of tokens - reducing gas costs for large transfers.

### Requirement for Security

### Fungible and Non-Fungible Token Differences

Although handled within the same contract, CEP-85 allows for both fungible and non-fungible tokens. These tokens are entirely separate within the contract instance, recorded in their associated dictionaries.

## Drawbacks and Alternatives

The major drawback to this standard is the potential for feature bloat and inefficiencies if the contract is forced to account for excessive, disparate data.

The simplest alternative would be the use of separate CEP-18 and CEP-78 token standard contracts with a custom contract to handle any crossover actions.

## Prior Art
[Ethereum request for comment 1155](https://eips.ethereum.org/EIPS/eip-1155) established the concept of a Multi-Token standard and can be considered the most impactful prior art.

## Unresolved Questions
Through our continuous iterative process we aim to iron out inefficiencies.

While CEP-18 and CEP-78 are well established standards, the combination of both types of token in a single contract may lead to unforeseen issues. We will listen to community feedback and refine over time.
