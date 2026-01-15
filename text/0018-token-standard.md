# Casper Payments: Casper Token Standard

## Summary
A standard interface for fungible tokens on the Casper blockchain. The standard
is inspired by the ERC-20 standard from Ethereum and is designed to be easy to
implement and use. The standard defines a set of entry points, events, error
codes, and storage structures that all CEP-18 tokens must implement.

## Prior art
We point at ERC-20 a the main source of influence: https://eips.ethereum.org/EIPS/eip-20

## Specification

The CEP-18 token is defined by 4 components:
- entrypoints,
- events,
- error codes,
- storage structure.

Below definitions use Rust syntax, but they are not Rust specific.

### Entrypoints interface

Contracts implementing this standard must expose the following entry points:

```rust
pub trait CEP18Interface {
    /// Returns the token name.
    fn name(&self) -> String;

    /// Returns the token symbol.
    fn symbol(&self) -> String;

    /// Returns the number of token decimals.
    fn decimals(&self) -> u8;

    /// Returns the total supply of tokens.
    fn total_supply(&self) -> U256;

    /// Returns the amount of tokens given address holds.
    /// The `account` is the address of the token holder.
    fn balance_of(&self, account: Key) -> U256;

    /// Returns the amount allowed to spend.
    /// The `owner` is the address of the token holder.
    /// The `spender` is the address of the account that is allowed to spend
    /// the tokens on behalf of the owner.
    fn allowance(&self, owner: Key, spender: Key) -> U256;

    /// Transfer tokens from the direct function caller to the `recipient`.
    /// The caller is the sender of the tokens.
    /// The `recipient` is the address of the account that will receive the tokens.
    /// The `amount` is the number of tokens to transfer.
    /// The transfer should fail:
    /// - if the caller does not have enough tokens,
    /// - if the recipient is the owner itself.
    fn transfer(&mut self, recipient: Key, amount: U256);

    /// Transfer tokens from `owner` address to the `recipient` address if required.
    /// The caller is the spender of the tokens.
    /// The `owner` is the address of the token holder.
    /// The `recipient` is the address of the account that will receive the tokens.
    /// The `amount` is the number of tokens to transfer.
    /// The transfer should fail:
    /// - if the recipient is the owner itself,
    /// - if the owner does not have enough tokens,
    /// - if the caller does not have enough allowance.
    fn transfer_from(&mut self, owner: Key, recipient: Key, amount: U256);

    /// Allow other address to transfer caller's tokens.
    /// The `spender` is the address of the account that is allowed to spend
    /// the tokens on behalf of the caller.
    /// The `amount` is the number of tokens to approve.
    fn approve(&mut self, spender: Key, amount: U256);

    /// Atomically decreases the allowance granted to spender by the caller.
    /// If the `decr_by` is greater than the current allowance, the allowance
    /// is set to zero.
    /// The `spender` is the address of the account that is allowed to spend
    /// the tokens on behalf of the caller.
    /// The `decr_by` is the amount by which the allowance should be decreased.
    fn decrease_allowance(&mut self, spender: Key, decr_by: U256);

    /// Atomically increases the allowance granted to spender by the caller.
    /// If the sum of the current allowance and `inc_by` is greater than the
    /// maximum value of U256, the allowance is set to the maximum value of U256.
    /// The `spender` is the address of the account that is allowed to spend
    /// the tokens on behalf of the caller.
    /// The `inc_by` is the amount by which the allowance should be increased.
    fn increase_allowance(&mut self, spender: Key, inc_by: U256);
}
```

### Events interface

Changes to the token state should be communicated to the outside world using events.
Events should be emitted using [Casper Event Standard](https://github.com/make-software/casper-event-standard).

Events definition:
```rust
pub enum CEP18Event {
    /// An event emitted when a mint operation is performed.
    Mint {
        /// The recipient of the minted tokens.
        recipient: Key,
        /// The amount of tokens minted.
        amount: U256
    },

    /// Emitted when the token is burned.
    Burn {
        /// The owner of the tokens that are burned.
        owner: Key,
        /// The amount of tokens burned.
        amount: U256
    },

    /// An event emitted when an allowance is set.
    SetAllowance {
        /// The owner of the tokens.
        owner: Key,
        /// The spender that is allowed to spend the tokens.
        spender: Key,
        /// The allowance amount.
        allowance: U256
    },

    /// An event emitted when an allowance is increased.
    IncreaseAllowance {
        /// The owner of the tokens.
        owner: Key,
        /// The spender that is allowed to spend the tokens.
        spender: Key,
        /// The final allowance amount.
        allowance: U256,
        /// The amount by which the allowance was increased.
        inc_by: U256
    },

    /// An event emitted when an allowance is decreased.
    DecreaseAllowance {
        /// The owner of the tokens.
        owner: Key,
        /// The spender that is allowed to spend the tokens.
        spender: Key,
        /// The final allowance amount.
        allowance: U256,
        /// The amount by which the allowance was decreased.
        decr_by: U256
    },

    /// An event emitted when a transfer is performed.
    Transfer {
        /// The sender of the tokens.
        sender: Key,
        /// The recipient of the tokens.
        recipient: Key,
        /// The amount of tokens transferred.
        amount: U256
    },

    /// An event emitted when a transfer_from is performed.
    TransferFrom {
        /// The spender that is allowed to spend the tokens.
        spender: Key,
        /// The sender of the tokens.
        owner: Key,
        /// The recipient of the tokens.
        recipient: Key,
        /// The amount of tokens transferred.
        amount: U256
    }
}
```

### Error Codes interface

The CEP-18 token contract should revert with the following error codes
when the appropriate conditions are met:

```rust
pub enum CEP18Error {
    /// Spender does not have enough balance.
    InsufficientBalance = 60001,
    /// Spender does not have enough allowance approved.
    InsufficientAllowance = 60002,
    /// The user cannot target themselves.
    CannotTargetSelfUser = 60003,
}
```

### Storage inteface

Querying the token state requires direct access to the contract storage. For
this reason, the layout of the CEP-18's contract storage is also part of the
standard.

#### Simple values

All the below simple values are stored in the contract's named keys under
following names:

- The name of the token is stored under the key `name` with the type `String`.
- The symbol of the token is stored under the key `symbol` with the type `String`.
- The number of decimals is stored under the key `decimals` with the type `u8`.
- The total supply of tokens is stored under the key `total_supply` with the type `U256`.

#### Balances

Balances are stored in the dictionary under the key `balances`. It is a
key-value storage where:
- The key is the account address of the token holder.
- The value is the amount of tokens held by the account.

The key is created using following algorithm: 
- The account address is converted to bytes using standard `CLType` encoding.
- The bytes are encoded to string using base64 encoding.

An example balance key generation implementation:

```rust
use base64::prelude::{Engine, BASE64_STANDARD};

fn balance_key(account: Key) -> String {
    let preimage = key.to_bytes().unwrap();
    BASE64_STANDARD.encode(preimage)
}
```

The value is stored as a `U256` value.

#### Allowances

Allowances are stored in the dictionary under the key `allowances`.
It is a key-value storage where:
- The key is a pair of two account addresses: the owner and the spender.
- The value is the amount of tokens that the spender is allowed to spend on behalf of the owner.

The key is created using following algorithm:
- The owner and spender addresses are converted to bytes using standard `CLType` encoding.
- The bytes are concatenated into a single bytes vector.
- The concatenated value is hashed using blake2b hashing algorithm.
- The hash is encoded to string using hex encoding.

An example balance key generation implementation:

```rust
fn allowance_key(owner: Key, spender: Key) -> String {
    let mut preimage = Vec::new();
    preimage.extend_from_slice(owner.to_bytes().unwrap());
    preimage.extend_from_slice(spender.to_bytes().unwrap());
    let hash_bytes = runtime::blake2b(preimage)
    hex::encode(hash_bytes)
}
```

## Exisiting implementations

The CEP-18 token implementations are available under the following links:
- https://github.com/casper-ecosystem/cep18
- https://github.com/odradev/odra/blob/HEAD/modules/src/cep18_token.rs
