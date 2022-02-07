# Casper Payments: Casper Token Standard

## Summary
A standard interface for the CSPR and the custom made tokens.

## Motivation
Having a token standard enables other projects to generalized on top it and makes
integration easy, especially for wallets, DAOs and DeFi products like decentralized
exchanges and liquidity pools. Defining a token standard before releasing
the mainnet gives use the unique opportunity to implement this standard for our
native platform token CSPR. It allows developers to reduce amount of code, 
because it is no longer required to handle 2 cases: one for native
token and the other for a tokens implementing current token standard. 

In the design we decided to get away from the purse based model and use 
the account based model. As a main inspiration we point at ERC-20 standard
known from Ethereum. It is the most widely adopted standard in the blockchain
space, successfully ported outside of Ethereum. Thanks to the unique ability
of Casper to execute code in the account's context, the main drawback of ERC-20,
which is the "approve and call" pattern is no longer a problem.

## Guide-level explanation
### Interface
The token contract should implement following endpoints.

#### name
Returns the token name.
```rust
fn name() -> String
```

#### symbol
Returns the token symbol.
```rust
fn symbol() -> String
```

#### decimals
Returns the number of token decimals.
```rust
fn decimals() -> u8
```

#### balance_of
Returns the amount of tokens given address holds.
```rust
fn balanceOf(address: Address) -> U512
```

#### batch_balance_of
Returns the amounts of tokens given a list of addresses.
```rust
fn batch_balance_of(addresses: Vec<Address>) -> Vec<U512>
```

#### transfer
Transfer tokens from the direct function caller to the `recipient`.
```rust
fn transfer(recipient: Address, amount: U512)
```

#### batch_transfer
Transfer tokens to multiple recipients.
```rust
fn batch_transfer(recipient_and_amount_list: Vec<(Address, U512)>)
```

#### approve
Allow other address to transfer caller's tokens.
```rust
fn approve(spender: Address, amount: U512)
```

#### batch_approve
Allow other addresses to transfer caller's tokens.
```rust
fn batch_approve(spender_and_amount_list: Vec<(Address, U512)>)
```

#### allowance
Returns the amount allowed to spend.
```rust
fn allowance(owner: Address, spender: Address) -> U512
```

#### batch_allowance
Returns the amounts allowed to spend for given addresses.
```rust
fn allowance(owner_spender_list: Vec<(Address, Address)>) -> Vec<U512>
```

#### transfer_from
Transfer tokens from `onwer` address to the `recipient` address if required
amount was approved before to be spend by the direct caller.
The operation should decrement approved amount.
```rust
fn transfer_from(owner: Address, recipient: Address, amount: U512)
```

#### batch_transfer_from
Transfer tokens from `onwer` address to the multiple `recipients` addresses if required
amount was approved before to be spend by the direct caller.
The operation should decrement approved amount.
```rust
fn batch_transfer_from(owner: Address, recipient_amount_list: Vec<(Address, U512)>)
```

### Compare to ERC-20
While very similar to ERC-20, this standard is a bit different:
1. Methods: `name`, `symbol` and `decimals` are required.
2. Names of arguments are part of the standard.
3. Added batch versions of methods.

## Reference-level explanation
### Custom tokens
We have successfully tested the implementation of ERC-20-based token.
Code is available at https://github.com/CasperLabs/erc20. 

### Casper Token
Currently Casper Token implements purse-based model. To support presented
standard reimplementation of Casper Token (Mint) is required. In addition 
Casper Token should be deployed at a know Address.

It should be possible to implement host-side interface, that manipulates
the memory directly, so CSPR transfers are as fast as possible.

## Rationale
In this section we compare purse-based model and account-based model.

Purse-based model allows accounts and contracts on creation of object
called `purses`. Each purse can have it's own balance. Each purse have
unique token access, that allows for tokens spending. This token access 
has to passed to contracts to send tokens.

### Scenario
Let's consider the most widely used scenario of interaction between two
contracts: sending tokens from contract to contract. In this example we
assume that:
1. `Oracle` contract is already deployed and can provide a dollar price for tokens.
2. `TokenEx` contract is deployed.
3. `Vault` contract is deployed.
4. `Source` contract is deployed.
5. Goal is to transfer 10 dollars worth of Ex tokens into the `Vault` from the `Source`.
6. Scenario starts by calling `action` on `Source`.

Below code is pseudo-code.

### Purse-based Model.
In this example `TokenEx` implements purse-based model.
```rust
[#casper_contract]
mod Source {

    [#casper_method]
    fn action(max_amount_of_tokens_allowed: U512) {
        // Read purse from the local contract's memory.
        let main_purse: URef = runtime::get_key("contracts_main_purse_for_ex_tokens");

        // Call the Token contract to transfer tokens into the new purse.
        // That's important, so the `Vault` contract is not exposed to the whole balance.
        let intermediate_purse: URef = 
            runtime::call_contract("token_ex_address", "transfer_to_new_purse", runtime_args!{
                from => main_purse,
                amount => max_amount_of_tokens_allowed
            });

        // Deposit tokens.
        runtime::call_contract("vault_address", "deposit_10_dollars", runtime_args!{
            ex_purse => intermediate_purse
        });

        // Transfer back the remaining from intermediate_purse to have all
        // the Ex tokens on one purse.
        runtime::call_contract("token_ex_address", "transfer_all", runtime_args!{
            from => intermediate_purse,
            to => main_purse
        });
    }
}

[#casper_contract]
mod Vault {

    [#casper_method]
    fn deposit_10_dollars(ex_purse: URef) {
        let main_purse: URef = runtime::get_key("contracts_main_purse_for_ex_tokens");
        let price = runtime::call_contract("oracle_address", "ex_price");
        let expected_amount = 10.0 / price;
        runtime::call_contact("token_ex_address", "transfer", runtime_args!{
            from => ex_purse,
            to => main_purse,
            amount => expected_amount
        });
    }
}
```

### Account-based Model.
In this example `TokenEx` implements account-based model.
```rust
[#casper_contract]
mod Source {

    [#casper_method]
    fn action(max_amount_of_tokens_allowed: U512) {
        // Approve tokens.
        runtime::call_contract("token_ex_address", "approve", runtime_args!{
            spender => "vault_address",
            amount => max_amount_of_tokens_allowed
        });

        // Deposit tokens.
        runtime::call_contract("vault_address", "deposit_10_dollars", runtime_args!{});
    }
}

[#casper_contract]
mod Vault {

    [#casper_method]
    fn deposit_10_dollars() {
        let price = runtime::call_contract("oracle_address", "ex_price");
        let expected_amount = 10.0 / price;
        runtime::call_contact("token_ex_address", "transferFrom", runtime_args!{
            from => runtime::get_caller(),
            to => "vault_address",
            amount => expected_amount
        });
    }
}
```


Account-based model is better because:
1. It doesn't require contracts and accounts to maintain purses in their 
   named keys space.
2. Transferring tokens in a purse-based model creates a lot of empty purses, that 
   probably will never be reused. The solution for that might some sort of 
   garbage collector, but that's just not a problem with the account-based model.
3. In most cases accounts and contracts will want to have all the tokens of one
   type in a single purse. That makes accounting much easier (also much cheaper).
   Account-based model gives that by the design.
4. Most of the platforms have token defined that way. Following that path,
   makes it much easier for developers to implement smart contract, because that's
   what they already know.
5. It will allow for much better Solidity portability via the Caspiler transpiler. 

### Approve and Call Problem
On Ethereum it is problematic for accounts to interact with contracts, that require
tokens. The account has to first call the `approve` function, wait for it, to
be executed and only then call desired contract's method.

On Casper platform contracts can provide helper functions for accounts, that are executed
in the account context. That helper function can easily aggregate multiple approves
and contract calls into one call.

## Drawbacks and alternatives
The alternative is to:
1. Keep current implementation of Mint and wait for more feedback on purse-based model.
2. Do not define any standard and see what would the community do.

## Prior art
We point at ERC-20 a the main source of influence: https://eips.ethereum.org/EIPS/eip-20

## Unresolved questions
It is useful if the interaction with contracts generates events, that describe 
what happened. Currently the Casper platform doesn't have that features, 
so this standard will have to be updated in a future, when the shape of the event 
functionality is known.

The standard doesn't cover the upgradeability story. Maybe it should?

