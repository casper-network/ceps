# Enhanced NFT standard

## Design goals
- DApp developer attempting to create an NFT contract should be able to install the contract as is,
  configured for the specific builtin behavior they want their NFT contract instance to have. Must work out of the box.
- Reference implementation must be straightforward, clear, and obvious.
- Externally observable association between `Accounts` and/or `Contracts` and `NFT`s they "own".
- Should be well documented with exhaustive tests that prove all possible combinations of defined behavior work as intended.
- Must be entirely self-contained within a singular repo, this includes the all code, all tests
  all relevant Casperlabs provided SDKs, and all relevant documentation.
- Must support mainstream expectations about common NFT conventions.
- A given NFT contract instance must be able to choose when created if it is using a Metadata schema conformant with existing community standards or a specific custom schema which they provide.
- A NFT contract instance must validate provided metadata against the specified metadata schema for that contract.
- Standardized session code to interact with an NFT contract instance must be usable as is, so that a given DApp developer doesn't have to write any Wasm producing logic for normal usage of NFT contract instances produced by this contract.

## Features and usage

### Modalities

The enhanced NFT implementation supports various 'modalities' which dictate the behavior of a specific instance of a
contract. Modalities represent the common expectations around contract usage and behavior.
The following section discusses the currently implemented modalities and illustrates the significance of each.

#### Ownership

This modality specifies the behavior regarding ownership of NFTs and also specifies whether the ownership
of the NFT can change over the lifetime of the contract. There are three modes:

1. `Minter`: `Minter` mode is where the ownership of the newly minted NFT is attributed to the minter of the NFT and cannot be specified by the minter. In the `Minter` mode the owner of the NFT will not change and thus cannot be transferred to another entity.
2. `Assigned`: `Assigned` mode is where the owner of the newly minted NFT must be specified by the minter of the NFT. In this mode, the assigned entity can be either minter themselves or a separate entity. However, similar to the `Minter` mode, the ownership in this mode cannot be changed and NFTs minted in this mode cannot be transferred from one entity to another.
3. `Transferable`: In the `Transferable` mode the owner of the newly minted NFT must be specified by the minter. However, in the `Transferable` mode, NFTs can be transferred from the owner to another entity.

In all the three mentioned modes, the owner entity is currently restricted to `Accounts` on the Casper network.

**Note**: In the `Transferrable` mode, it is possible to transfer the NFT to an `Account` that does not exist.

This `Ownership` mode is a required installation parameter and cannot be changed once the contract has been installed.
The mode is passed in as `u8` value to the `"ownership_mode"` runtime argument.

| Mode         | u8  |
|--------------|-----|
| Minter       | 0   |
| Assigned     | 1   |
| Transferable | 2   |

The ownership mode of a contract can be determined by querying the `ownership_mode` entry within the contract's `NamedKeys`.

#### NFTKind

The `NFTKind` modality specifies the commodity that NFTs minted by a particular contract will represent. Currently, the `NFTKind` modality does not alter or govern the behavior of the contract itself
and only exists to specify the correlation between on-chain data and off-chain items. There are three different variatations of the `NFTKind` mode.

1. `Physical`: The NFT represents a real-world physical item e.g a house.
2. `Digital`: The NFT represents a digital item, e.g a unique JPEG or a digital art.
3. `Virtual`: The NFT is the virtual representation of a physical notion, e.g a patent or copyright.

The `NFTKind` mode is a required installation parameter and cannot be changed once the contract has been installed.
The mode is passed in as a `u8` value to `nft_kind` runtime argument.

| NFTKind  | u8  |
|----------|-----|
| Physical | 0   |
| Digital  | 1   |
| Virtual  | 2   |


#### Minting

The minting mode governs the behavior of contract when minting new tokens. The minting modality provides two options:

1. `Installer`: This mode restricts the ability to mint new NFT tokens only to the installing account of the NFT contract.
2. `Public`: This mode allows any account to mint NFT tokens.

This modality is an optional installation parameter and will default to the `Installer` mode if not provided. However, this
mode cannot be changed once the contract has been installed. The mode is set by passing a `u8` value to the `minting_mode` runtime argument.

| MintingMode | u8  |
|-------------|-----|
| Installer   | 0   |
| Public      | 1   |


#### Modality Conflicts

The implemented modalities have no conflicting behavior.


### Usage

#### Installing the contract.

The `main.rs` file within the contract provides the installer for the NFT contract. Users can compile the contract to Wasm using the `make build-contract` with the provided Makefile.
The `call` method will install the contract with the necessary entrypoints and call `init()` entry point to allow the contract to self initialize and setup the necessary state to allow for operation,
The following are the required runtime arguments that must be passed to the installer session code to correctly install the NFT contract.

* `"collection_name":` The name of the NFT collection, passed in as a `String`. This parameter is required and cannot be changed post installation.
* `"collection_symbol"`: The symbol representing a given NFT collection, passed in as a `String`. This parameter is required and cannot be changed post installation.
* `"total_token_supply"`: The total number of NFTs that a specific instance of a contract will mint passed in as a `U64` value. This parameter is required and cannot be changed post installation.
* `"ownership_mode"`: The [`OwnershipMode`](#ownership) modality that dictates the ownership behavior of the NFT contract. This argument is passed in as a `u8` value and is required at the time of installation.
* `"nft_kind"`: The [`NFTKind`](#nftkind) modality that specifies the off-chain items represented by the on-chain NFT data. This argument is passed in as a `u8` value and is required at the time of installation.
* `"json_schema"`: The JSON schema for the NFT tokens that will be minted by the NFT contract passed in as a `String`. This parameter is required and cannot be changed post installation.

The following are the optional parameters that can be passed in at the time of installation.

* `"minting_mode"`: The [`MintingMode`](#minting) modality that dictates the access to the `mint()` entry-point in the NFT contract. This is an optional parameter that will default to restricting access to the installer of the contract. This parameter cannot be changed once the contract has been installed.
* `"allow_minting"`: The `"allow_minting"` flag that allows the installer of the contract to pause the minting of new NFTs. The `allow_minting` is a boolean toggle which allows minting when `true`. If not provided at install the toggle will default to `true`. This value can be changed by the installer by calling the `set_variables()` entrypoint.

##### Example deploy

The following is an example of deploying the installation of the NFT contract via the Rust Casper command client.

```rust
casper-client put-deploy -n http://localhost:11101/rpc --chain-name "casper-net-1" --payment-amount 500000000000 -k ~/casper/casper-node/utils/nctl/assets/net-1/nodes/node-1/keys/secret_key.pem --session-path ~/casper/enhanced-nft/contract/target/wasm32-unknown-unknown/release/contract.wasm --session-arg "collection_name:string='enhanced-nft-1'" --session-arg "collection_symbol:string='ENFT-1'" --session-arg "total_token_supply:u256='10'" --session-arg "ownership_mode:u8='0'" --session-arg "nft_kind:u8='1'" --session-arg "json_schema:string='nft-schema'" --session-arg "allow_minting:bool='true'" 
```

#### Utility session code

Certain entry points in use by the current implementation of the NFT contract require session code to accept return values passed by the contract over the Wasm boundary.
In order to help with the installation and use of the NFT contract, session code for such entry points has been provided. It is recommended that
users and d-app developers attempting to engage with the NFT contract do so with the help of the provided utility session code. The session code can be found in the `client`
folder within the project folder.

| Entry point name | Session code                  |
|------------------|-------------------------------|
| `"mint"`         | `client/mint_session`         |
| `"balance_of"`   | `client/balance_of_session`   |
| `"get_approved`  | `client/get_approved_session` |
| `"owner_of"`     | `client/owner_of_session`     |


## Test Suite and specification.

The expected behavior of the NFT contract implementation is asserted by its test suite found in the `tests` folder.
The test suite and the corresponding unit tests comprise the specification around the contract and outline the expected behaviors
of the NFT contract across the entire range of possible configurations (i.e modalities and toggles like allow minting). The test suite
ensures that as new modalities are added and current modalities are extended no regressions and conflicting behaviors are introduced.
The test suite also asserts the correct working behavior of the utility session code provided in the client folder. The tests can be run
by using the provided `Makefile` and running the `make test` command.

