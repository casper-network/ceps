# CEP47 - Casper NFT Protocol

## Summary

CEP PR: [casper-network/ceps#0047](https://github.com/casper-network/ceps/pull/47)

The NFT smart contact protocol designed for Casper platform. 

## Motivation

Casper ecosystem needs an NFT standard to enable further development of 
the NFT-based products. Standards that originates from Ethereum ecosystem 
are not the best choice for the Casper platform, because of the Casper 
unique features. 

CEP47 takes the full advantage of the URef feature to optimize the smart
contracts interactions. It also uses the Groups feature to optimize gas costs.

## Guide-level explanation

CEP47 is expected to be a single contract setup.

The main identifier of the CEP47 contract should be the `ContractPackageHash`.

### Data model

CEP47 contract allows to have an NFT token that is represented by:
- `name` - the global name of the whole contract,
- `symbol` - the global symbol of the wole contract,
- `uri` - the link to, or a hash of a file with off-chain NFT description.      

The contract can issue multiple tokens. Each token can have its own `uri`,
that describes the token instance.

Each account represented by `PublicKey` can own multiple tokens. Information
on who owns what is stored inside the contract.

Each token can be `detached` for the owner. It means the ownership of the token
is no longer attached to the `PublicKey`, but to the `URef`. The `URef` can be
shared with contracts. Contracts or accounts can also `attach` the token back
to the given `PublicKey`. Information on which URef represents which token
is stored inside the contract. `detach` and `attach` enables contracts to
own and modify ownership of tokens.

### Endpoints

CEP47 standard defines a list of endpoints that allows to issue and manage NFTs.

Endpoints can be grouped into following topics:
#### Metadata

Endoints:
- `name`
- `symbol`
- `uri`
- `total_supply`
- `balance_of`
- `token_uri`
- `owner_of`
- `tokens_of`
- `token_uref`
  
are ment to be used by another smart contracts. 

Above endpoints allows contract to get information about the NFT contract, token instances
and ownership of tokens.

#### Transfers

Endpoints:
- `transfer_token`
- `transfer_many_tokens`
- `trasnfer_all_tokens`

are ment to be used by the account context only.

Above endpoints provide the interface for transfering tokens from the sender account
to another account.

#### Mints

Endpoints:
- `mint_one`
- `mint_many`
- `mint_copies`

are ment to be used by both accounts and smart contracts.

Above endpoints allow to create new instances of NFTs. Those should be guarded
by the `minter_group` user group on the contract header level.

#### URef-based Endpoints

Endpoints:
- `detach`
- `attach`

are ment to be used by both accounts and smart contract.

Above endpoints allow to move the ownership of the single NFT from the
account owning it to the URef, aka `detach` the token. It improves and
simplifies the smart contract interactions. Such URef can be passed to
the another contract and even further to yet another contracts in the 
subcalls. The token ownership is passed along with URef and enables 
contracts to transfer the token without a need of interaction with 
the main NFT contract. The contract or the account can decide to `attach`
the token back to the given account. 

## Reference-level explanation

### Types

`String` type is used to represent `URI`.

`String` type is used to represent `TokenId`.

`U256` type is used to represent balances.

`PublicKey` type is used to represent the account. 

### User Groups

Following endpoints should be guarded by the `minter_group`:
- `mint_one`
- `mint_many`
- `mint_copies` 

### Endpoints

#### Total Supply

Returns the amount of issued NFTs.

```rust
EntryPoint::new(
    String::from("total_supply"),
    vec![],
    CLType::U256,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Name

Returns the name of the NFT contract.

```rust
EntryPoint::new(
    String::from("name"),
    vec![],
    CLType::String,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Symbol

Returns the symbol of the NFT contract.

```rust
EntryPoint::new(
    String::from("symbol"),
    vec![],
    CLType::String,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### URI

Returns the URI of the NFT contract.

It could be a hash of IPFS document, that contains more details of the NFT. 

```rust
EntryPoint::new(
    String::from("uri"),
    vec![],
    CLType::String,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Balance Of

Returns a number of NFT instances the `owner` owns.

```rust
EntryPoint::new(
    String::from("balance_of"),
    vec![Parameter::new("owner", CLType::PublicKey)],
    CLType::U256,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Token URI

Returns the URI of the NFT token instance.

Each token can have it's own unique URI.

It fails if the `token_id` doesn't exist.

```rust
EntryPoint::new(
    String::from("token_uri"),
    vec![Parameter::new("token_id", CLType::String)],
    Box::new(CLType::PublicKey),
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Owner Of

Returns an owner of the given `token_id`.

It returns `None` if the `token_id` doesn't have owner.

```rust
EntryPoint::new(
    String::from("owner_of"),
    vec![Parameter::new("token_id", CLType::String)],
    CLType::Option(Box::new(CLType::PublicKey)),
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Tokens Of

Return a list of tokens of the `owner`. 

```rust
EntryPoint::new(
    String::from("tokens_of"),
    vec![Parameter::new("owner", CLType::PublicKey)],
    CLType::List(Box::new(CLType::String)),
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Mint One

Mint a new NFT instance with the given `token_uri` and transfer it
to the `recipient` account.

```rust
EntryPoint::new(
    String::from("mint_one"),
    vec![
        Parameter::new("recipient", CLType::PublicKey),
        Parameter::new("token_uri", CLType::String),
    ],
    CLType::Unit,
    EntryPointAccess::groups(&["minter_group"]),
    EntryPointType::Contract,
)
```

#### Mint Many

Mint many new NFT instances with the given `token_uris` and transfer all
to the `recipient` account.

It allows to save gas when minting a list of unique tokens.

```rust
EntryPoint::new(
    String::from("mint_many"),
    vec![
        Parameter::new("recipient", CLType::PublicKey),
        Parameter::new("token_uris", CLType::List(Box::new(CLType::String))),
    ],
    CLType::Unit,
    EntryPointAccess::groups(&["minter_group"]),
    EntryPointType::Contract,
)
```

#### Mint Copies

Mint many new NFT instances, all with the same `token_uri` and transfer all
to the `recipient` account.

It allows to save gas when minting a list of identical tokens.

```rust
EntryPoint::new(
    String::from("mint_copies"),
    vec![
        Parameter::new("recipient", CLType::PublicKey),
        Parameter::new("token_uri", CLType::String),
    ],
    CLType::Unit,
    EntryPointAccess::groups(&["minter_group"]),
    EntryPointType::Contract,
)
```

#### Transfer Token

Transfer a single token from `sender` to `recipient`.

The `sender` should the owner of a `token_id`.

Contract should assert `sedner.to_account_hash()` to be equal to 
the `runtime::get_caller()`.

```rust
EntryPoint::new(
    String::from("transfer_token"),
    vec![
        Parameter::new("sender", CLType::PublicKey),
        Parameter::new("recipient", CLType::PublicKey),
        Parameter::new("token_id", CLType::String),
    ],
    CLType::Unit,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Transfer Many Tokens

Transfer many tokens from `sender` to `recipient`.

The `sender` should the owner of all `token_ids`.

Contract should assert `sedner.to_account_hash()` to be equal to 
the `runtime::get_caller()`.

It allows to save gas when sending many tokens.

```rust
EntryPoint::new(
    String::from("transfer_many_tokens"),
    vec![
        Parameter::new("sender", CLType::PublicKey),
        Parameter::new("recipient", CLType::PublicKey),
        Parameter::new("token_ids", CLType::List(Box::new(CLType::String)))
    ],
    CLType::Unit,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Transfer All Tokens

Transfer all tokens from `sender` to `recipient`.

Contract should assert `sedner.to_account_hash()` to be equal to 
the `runtime::get_caller()`.

It allows to save gas when sending all tokens.

```rust
EntryPoint::new(
    String::from("transfer_all_tokens"),
    vec![
        Parameter::new("sender", CLType::PublicKey),
        Parameter::new("recipient", CLType::PublicKey)
    ],
    CLType::Unit,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Detach

Transfer the ownership of the `token_id` NFT to the `URef` object
and return that object.

Contract should assert `owner.to_account_hash()` to be equal to 
the `runtime::get_caller()`.

```rust
EntryPoint::new(
    String::from("detach"),
    vec![
        Parameter::new("owner", CLType::PublicKey),
        Parameter::new("token_id", CLType::String)
    ],
    CLType::URef,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Attach

Transfer the ownership of the token from the `URef` object back
to the NFT contract and transfer it to the given `recipient`.

Contract should assert `owner.to_account_hash()` to be equal to 
the `runtime::get_caller()`.

```rust
EntryPoint::new(
    String::from("attach"),
    vec![
        Parameter::new("token_uref", CLType::URef)
        Parameter::new("recipient", CLType::PublicKey),
    ],
    CLType::Unit,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

#### Token Id

Returns a `token_id` of the given `token_uref`.

```rust
EntryPoint::new(
    String::from("token_uref"),
    vec![Parameter::new("token_uref", CLType::URef)],
    CLType::String,
    EntryPointAccess::Public,
    EntryPointType::Contract,
)
```

## Drawbacks

## Rationale and alternatives

The design fits the Casper v1.1.2. 

In the upcomming releases new features will be avaliable, and the standard
should adopt along them. Those features are: Local Keys and Call Stack.

## Prior art

NFT standards originate in Ethereum, which ERC721 is the most used one.
CEP47 solves the same problem as ERC721, but ERC721 could not be used
directly, as it fits EVM. Casper VM has different traits, so the Casper
NFT standard needs to fit those features. As a result, we achieved
a gas efficient design.

## Unresolved questions

## Future possibilities

