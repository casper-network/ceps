# Casper NFT Standard Proposal

Casper NFT Standard Proposal (Inspired by ERC-721 & adapted for the Casper ecosystem)

## Summary

CEP PR: [casperlabs/ceps#0095](https://github.com/casper-network/ceps/pull/95)

This standard defines a unified API for implementing non-fungible tokens (NFTs) on the Casper network. It introduces core functionality for tracking and transferring NFTs in a consistent and interoperable way.

We have taken into account use cases where NFTs are owned directly by individuals, as well as scenarios where ownership and transfers are managed by third-party entities such as marketplaces, wallets, or custodial services ("operators"). NFTs on Casper can represent ownership of both digital and physical assets, and the range of potential applications continues to grow:

- Physical assets — real estate, unique artwork 
- Digital collectibles — rare digital items, game assets, virtual companions 
- Obligations and non-traditional assets — debt instruments, legal contracts, responsibilities

Each NFT is uniquely identifiable, and as such, must be tracked individually. Unlike fungible tokens, NFTs are not interchangeable — every item is distinct, and this standard ensures that uniqueness and ownership are verifiable and transferable on-chain.

## Motivation

[motivation]: #motivation
A standard interface for non-fungible tokens (NFTs) on the Casper network enables wallets, marketplaces, decentralized applications, and other ecosystem tools to interact seamlessly with any compliant NFT contract. The lessons learned from Ethereum’s ERC-721 standard show that a consistent interface increases interoperability and speeds up adoption. On Casper, this standard will ensure that every NFT, whether tracking digital collectibles or physical assets, is uniquely identifiable and transferable.

## Objective

[objective]: #objective

The objective of this proposal is to define a Casper-compatible NFT standard that supports:

- Tracking ownership and metadata of NFTs. 
- Transferring NFTs safely between accounts. 
- Managing per-token approvals as well as operator-level approvals.
- (Optionally) exposing metadata (name, symbol, token URI) so that applications can easily display NFT details.

This proposal also addresses the backend indexing and tracking of NFT events (minting, transfer, approvals) for full visibility into asset histories.

## Core Casper NFT Interface

[core-casper-nft-interface]: #core-casper-nft-interface

The core interface is designed to address the basic functionality required to manage non-fungible tokens. It uses Casper’s types (e.g., Key, U256) and aims to capture the semantics of ERC-721, adjusted for Casper’s execution model.

```rust
#![no_std]

use casper_contract::contract_api::{runtime, storage};
use casper_types::{Key, URef, U256, ContractHash};

/// Enum defining events that should be emitted by NFT operations
#[derive(Debug)]
pub enum CEP95Event {
    /// Emitted when an NFT is minted (transfer - from=None)
    Mint { to: Key, token_id: U256 },

    /// Emitted when an NFT is transferred
    Transfer { from: Key, to: Key, token_id: U256 },
    
    /// Emitted when an NFT is burned (transfer - to=None)
    Burn { from: Key, token_id: U256 },

    /// Emitted when a specific NFT is approved to an account/contract
    Approval { owner: Key, spender: Key, token_id: U256 },
    
    /// Emitted when a specific NFT approval is revoked from an account/contract
    RevokeApproval { owner: Key, spender: Key, token_id: U256 },

    /// Emitted when an operator is approved for all NFTs of an owner
    ApprovalForAll { owner: Key, operator: Key },
    
    /// Emitted when an operator approval is revoked for all NFTs of an owner
    RevokeApprovalForAll { owner: Key, operator: Key },
}

/// Casper-compatible NFT interface
pub trait CEP95 {
    /// Returns the number of NFTs owned by a given account
    ///
    /// @param owner - The account to query.
    /// @return U256 - The number of NFTs owned.
    fn balance_of(&self, owner: Key) -> U256;

    /// Returns the owner of a specific NFT.
    ///
    /// @param token_id - The ID of the NFT.
    /// @return Option<Key> - The owner if it exists, else None.
    fn owner_of(&self, token_id: U256) -> Option<Key>;

    /// Transfers the ownership of an NFT and performs a recipient check.
    ///
    /// @param from - The current owner of the NFT.
    /// @param to - The new owner.
    /// @param token_id - The NFT ID.
    /// @param data - Optional payload to pass to a receiving contract.
    fn safe_transfer_from(&mut self, from: Key, to: Key, token_id: U256, data: Option<Vec<u8>>);

    /// Transfers the ownership of an NFT without checking the recipient contract.
    ///
    /// @param from - The current owner of the NFT.
    /// @param to - The new owner.
    /// @param token_id - The NFT ID.
    fn transfer_from(&mut self, from: Key, to: Key, token_id: U256);

    /// Approves another account to transfer a specific NFT.
    ///
    /// @param to - The account that will be granted approval.
    /// @param token_id - The NFT ID.
    fn approve(&mut self, spender: Key, token_id: U256);

    /// Revokes approval for a specific NFT.
    ///
    /// @param token_id - The NFT ID.
    fn revoke_approval(&mut self, token_id: U256);

    /// Gets the approved account for a specific NFT.
    ///
    /// @param token_id - The NFT ID.
    /// @return Option<Key> - Approved spender account if one exists.
    fn get_approved(&self, token_id: U256) -> Option<Key>;

    /// Enables operator approval for all of the caller's NFTs.
    ///
    /// @param operator - The operator address to be approved.
    fn approve_for_all(&mut self, operator: Key);

    /// Revokes operator approval for all of the caller's NFTs.
    ///
    /// @param operator - The operator address whose approval is to be revoked.
    fn revoke_approval_for_all(&mut self, operator: Key);

    /// Checks if an operator is approved to manage all NFTs of the owner.
    ///
    /// @param owner - The NFT owner's address.
    /// @param operator - The operator to check.
    /// @return bool - True if the operator is approved for all NFTs, false otherwise.
    fn is_approved_for_all(&self, owner: Key, operator: Key) -> bool;
}
```

## Metadata Extension

[metadata-extension]: #metadata-extension

The metadata extension is OPTIONAL for Casper NFT smart contracts. This extension allows your smart contract to be interrogated for its collection name, symbol, and a URI pointing to metadata describing the assets represented by your NFTs.

### Contract Metadata Interface: CEP95ContractMetadata

```rust
pub trait CEP95ContractMetadata {
    /// Returns a descriptive name for a collection of NFTs.
    fn name(&self) -> String;

    /// Returns a short symbol or abbreviation for the NFT collection.
    fn symbol(&self) -> String;
}
```

### Token Offchain Metadata Interface: CEP95TokenOffchainMetadata

```rust
/// Optional interface for retrieving on-chain token metadata
pub trait CEP95TokenOffchainMetadata {
    /// Returns a URI pointing to metadata for a specific token.
    ///
    /// @param token_id - The NFT ID.
    /// @return Option<String> - A URI pointing to JSON metadata if available.
    fn token_uri(&self, token_id: U256) -> Option<String>;
}
```

Note: In this approach, the on-chain function returns a URI that points to the off-chain metadata resource rather than storing the metadata directly on-chain.

### Token Onchain Metadata Interface: CEP95TokenOffchainMetadata

```rust
#[derive(Debug)]
pub enum CEP95MetadataEvent {
    /// Emitted whenever on-chain metadata for a token is created or updated.
    ///
    /// @param token_id - The ID of the NFT whose metadata was updated
    /// @param metadata - The raw metadata (e.g., JSON bytes)
    MetadataUpdate {
        token_id: U256,
        metadata: Vec<(String, String)>,
    },
}

/// Optional interface for retrieving on-chain token metadata
pub trait CEP95TokenOnchainMetadata {
    /// Returns raw metadata for a given token ID.
    ///
    /// @param token_id - The NFT ID.
    /// @return Option<Vec<(String, String)>> - Encoded metadata blob (e.g., JSON-encoded bytes, CBOR, etc.)
    fn token_metadata(&self, token_id: U256) -> Option<Vec<(String, String)>>;
    
    /// Stores raw metadata for a given token ID.
    /// - Optional convenience method for setting or updating on-chain metadata.
    ///
    /// @param token_id: The unique ID of the NFT.
    /// @param metadata: The raw bytes representing token metadata (JSON, etc.).
    fn set_token_metadata(&mut self, token_id: U256, metadata: Vec<(String, String)>);
}
```

Note: In this approach, the on-chain function `token_metadata` returns a `Vec<(String, String)>` that is a metadata resource stored directly on-chain.

### Example Metadata Schema

The following conventional JSON schema describes the expected format of metadata referenced by the URI returned by `token_uri` or raw bytes returned by `token_metadata` method:

```json
{
    "title": "Asset Metadata",
    "type": "object",
    "properties": {
        "name": {
            "type": "string",
            "description": "Identifies the asset to which this NFT represents"
        },
        "description": {
            "type": "string",
            "description": "Describes the asset to which this NFT represents"
        },
        "asset_uri": {
            "type": "string",
            "description": "A URI pointing to a resource with mime type image/* representing the asset to which this NFT represents. Consider making any images at a width between 320 and 1080 pixels and aspect ratio between 1.91:1 and 4:5 inclusive."
        }
    }
}
```

## Event Structure

[event-structure]: #event-structure

### CEP95Event Enum

Efficient event tracking is essential for indexing and off-chain applications. Your NFT standard emits events for transfers, approvals, and operator approvals.

```rust
#[derive(Debug)]
pub enum CEP95Event {
    Mint { to: Key, token_id: U256 },
    Transfer { from: Key, to: Key, token_id: U256 },
    Burn { from: Key, token_id: U256 },
    Approval { owner: Key, spender: Key, token_id: U256 },
    RevokeApproval { owner: Key, spender: Key, token_id: U256 },
    ApprovalForAll { owner: Key, operator: Key },
    RevokeApprovalForAll { owner: Key, operator: Key },
}
```

- `Mint` is emitted when an NFT is minted (transfer - from=None).
- `Transfer` is emitted when an NFT is transferred.
- `Burn` is emitted when an NFT is burned (transfer - to=None).
- `Approval` is emitted when a single NFT is approved to be transferred by another account/contract.
- `RevokeApproval` is emitted when a specific NFT approval is revoked from an account/contract.
- `ApprovalForAll` is emitted when an operator’s permissions for all NFTs are enabled.
- `RevokeApprovalForAll` is emitted when an operator’s permissions for all NFTs are revoked.

### CEP95Event Enum

```rust
#[derive(Debug)]
pub enum CEP95MetadataEvent {
    /// Emitted whenever on-chain metadata for a token is created or updated.
    ///
    /// @param token_id - The ID of the NFT whose metadata was updated
    /// @param metadata - The raw metadata (e.g., JSON bytes)
    MetadataUpdate {
        token_id: U256,
        metadata: Vec<(String, String)>,
    },
}
```

- `MetadataUpdate` is emitted when an on-chain metadata for a token is created or updated.

## Usage and Implementation

[usage-and-implementation]: #usage-and-implementation

### Minting an NFT

- **Minting Process**: When minting a new NFT, the smart contract should emit a `Transfer` event with `from: None` and `to: Some(account_hash)`. 
- **State Updates**: The contract will record the new NFT’s `token_id` with its owner and (optionally) associate metadata (via a dictionary or another persistent storage mechanism).

### Transferring an NFT

- **Safe vs. Unsafe Transfers**: Use `safe_transfer_from` when transferring to a contract (to allow the contract to confirm it can handle NFTs) and `transfer_from` for basic transfers.
- **Event Emission**: Both methods should update the owner mapping and emit a `Transfer` event.

### Approval Mechanisms

- **Per-token Approval**: The `approve` function enables an account to transfer a specific NFT. If needed, revoke_approval clears that approval.
- **Operator Approvals**: The `approve_for_all` and `revoke_approval_for_all` functions allow a user to delegate transfer rights over their entire collection, with an event emitted to signal the change.


## Backend Indexing and NFT Metadata Tracking

[backend-indexing-and-nft-metadata-tracking]: #backend-indexing-and-nft-metadata-tracking

### Tracking NFT Events

Backend indexers or off-chain processors should listen for NFT-related events:

- **Minting**: Detect a `Mint` event. Extract the `token_id` and `to` address.
- **Transfers and Approvals**: Capture changes in ownership or approvals by parsing `Transfer`, `Approval`, `RevokeApproval`, `ApprovalForAll` and `RevokeApprovalForAll` events.
- **Burning**: Detect a `Burn` event. Extract the `token_id` and `from` address.

Optional:
- **Onchain Metadata**: Detect `MetadataUpdate` event. Extract the `token_id` and `metadata` bytes.

### Querying NFT Offchain Metadata

Once an event (such as minting) is detected:

1. **Extract `token_id`**: From the event, identify the newly minted NFT. 
2. **Query token URI**: Invoke or query the contract’s `token_uri(token_id)` function using Casper’s JSON-RPC methods (e.g., `state_get_item` or `state_get_dictionary_item`) or a read-only contract call. 
3. **Fetch Off-Chain Metadata**: With the obtained URI, perform an HTTP/IPFS fetch to retrieve the JSON metadata. Parse the fields (name, description, asset_uri, etc.). 
4. **Indexing**: Store the NFT’s `token_id`, `owner`, and `metadata` in your database for efficient querying and display in applications.


## Conclusion

[conclusion]: #conclusion

The Casper NFT Standard (inspired by ERC-721) aims to bring uniformity and interoperability to NFT development on Casper. It provides simple methods for handling tokens, approvals, and optional metadata, making it easier for wallets, marketplaces, and indexers to work with NFTs. This proposal helps build a connected and thriving NFT ecosystem on Casper.
## Appendix

[appendix]: #appendix

### Changelog

- v1.0: Initial draft including CEP95 and CEP95Metadata interfaces, event structures, and backend indexing guidelines.

### References

- [ERC-721 Standard on Ethereum](https://eips.ethereum.org/EIPS/eip-721)
- [Casper Documentation (for contract APIs and types)](https://docs.casper.network/concepts/smart-contracts)
- [Example proposals and CEPs from the Casper community](https://github.com/make-software/casper-ceps/tree/master/text)
