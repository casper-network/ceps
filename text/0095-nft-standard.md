# Casper NFT Standard Proposal

Casper NFT Standard Proposal (Inspired by ERC-721 & adapted for the Casper ecosystem)

## Summary

CEP PR: [casperlabs/ceps#0095]

This standard provides a non-fungible token (NFT) contract API that is consistent with the existing [CEP-18](https://github.com/casper-network/ceps/blob/master/text/0018-token-standard.md) fungible token standard. It aims to replace [CEP-47](https://github.com/casper-network/ceps/blob/master/text/0047-casper-nft-protocol.md) and [CEP-78](https://github.com/casper-network/ceps/blob/master/text/0078-enhanced-nft-standard.md), which have flaws that complicate their support in the ecosystem. CEP-47 lacks operator approval functionality, which makes it less convenient to integrate with NFT marketplaces. CEP-78 rather describes a configurable NFT contract that can modify its behavior to various needs, rather than a plain NFT contract API.

This standard is aligned with Ethereum's [ERC-721](https://eips.ethereum.org/EIPS/eip-721), but makes adjustments relevant for the Casper Ecosystem. Similarly to ERC-721, this standard can be used to represent a various range of tokenized assets:
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

- Tracking ownership of NFTs.
- Transferring NFTs safely between accounts and contracts.
- Managing per-token approvals as well as operator-level approvals.
- (Optionally) exposing metadata (name, symbol, token URI) that describes the underlying asset details.

This proposal takes into account a possibility of off-chain indexing of NFTs by specifying interfaces of the events that can be used to track the contract activity.
## Specification

[specification]: #specification

### Core Casper NFT Interface

[core-casper-nft-interface]: #core-casper-nft-interface

The core interface is designed to address the basic functionality required to manage non-fungible tokens. It uses Casper’s types (e.g., Key, U256) and aims to capture the semantics of ERC-721, adjusted for Casper’s execution model.

```rust
pub enum CEP95Event {
    /// Emitted when an NFT is minted
    Mint { to: Key, token_id: U256 },

    /// Emitted when an NFT is transferred
    Transfer { from: Key, to: Key, token_id: U256 },
    
    /// Emitted when an NFT is burned
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
    /// Returns an optional short symbol or abbreviation for the NFT collection.
    fn symbol(&self) -> Option<String>;

    /// Returns the number of NFTs owned by a given account or contract
    ///
    /// @param owner - The account to query.
    /// @return U256 - The number of NFTs owned.
    fn balance_of(&self, owner: Key) -> U256;

    /// Returns the owner of a specific NFT.
    ///
    /// @param token_id - The ID of the NFT.
    /// @return Option<Key> - The owner if it exists, else None.
    fn owner_of(&self, token_id: U256) -> Option<Key>;

    /// Performs a recipient check and transfers the ownership of an NFT.
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

    /// Approves another account or contract to transfer a specific NFT.
    ///
    /// @param to - The account that will be granted approval.
    /// @param token_id - The NFT ID.
    fn approve(&mut self, spender: Key, token_id: U256);

    /// Revokes approval for a specific NFT.
    ///
    /// @param token_id - The NFT ID.
    fn revoke_approval(&mut self, token_id: U256);

    /// Gets the approved account or contract for a specific NFT.
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

### Recommended Adoption of the Casper On-chain Contract Metadata Standard CEP-96

To improve the integrity, discoverability, and user experience within the Casper ecosystem, implementers of the CEP-95 NFT Standard are strongly encouraged to also adopt the **CEP-96 Casper On-chain Contract Metadata Standard**.

By implementing these metadata standard, CEP-95 contracts become easily identifiable, understandable, and trustworthy to end-users, explorers, wallets, and marketplaces. The adoption of the On-chain Contract Metadata Standard enhances the overall integrity and usability of the Casper blockchain ecosystem and helps build user trust.

We strongly recommend incorporating this metadata standard alongside CEP-95 implementations to contribute positively to ecosystem consistency, transparency, and user-friendliness.

### Recommended Storage for contract data (symbol, etc.)

According to the CEP-96 standard, it is recommended to store contract metadata as a form of contract level named keys using `Urefs`.

**Example Named Key Usage (High-Level)**

1. Install/Constructor Phase:
    ```rust
    let symbol_arg: Option<String> = runtime::get_named_arg("symbol");
    if let Some(symbol_value) = symbol_arg {
        // Create a new URef for the symbol and place it as a named key
        let symbol_uref = storage::new_uref(symbol_value);
        runtime::put_key("symbol", symbol_uref.into());
    }
    ```
2. Interface Method:
    ```rust
    #[no_mangle]
    pub extern "C" fn symbol() {
        let symbol: Option<String> = match runtime::get_key("symbol") {
            Some(key) => {
                let symbol_uref = key.into_uref().expect("symbol key is not a URef");
                storage::read(symbol_uref).expect("unable to read symbol from storage")
            }
            None => None,
        };
        runtime::ret(CLValue::from_t(symbol).unwrap_or_revert());
    } 
    ```

## Token Metadata Extension

[token-metadata-extension]: #token-metadata-extension

### Hybrid Approach for Token Metadata in CEP-95

[hybrid-approach-for-token-metadata-in-cep-95]: #hybrid-approach-for-token-metadata-in-cep-95

The CEP-95 standard provides a flexible, hybrid approach for NFT token metadata, allowing developers and consumers to leverage **both on-chain and off-chain metadata sources**.

- **On-chain metadata** (`token_metadata`):
  - This method allows direct storage of metadata within the blockchain. Metadata is stored as key-value pairs (`Vec<(String, String)>`), giving a clear yet flexible structure without enforcing a specific serialization format. When metadata is created or updated, the contract emits a `MetadataUpdate` event, ensuring that indexers and clients are immediately informed and can cache or display this data without additional lookups. 
- **Off-chain metadata** (`token_uri`):
  - Additionally, CEP-95 supports traditional off-chain metadata retrieval via a URI. Contracts can provide a URI linking to external JSON metadata (e.g., hosted on IPFS or HTTP servers). This off-chain approach reduces on-chain storage costs and allows easier updates of non-critical metadata fields.

Consumers, such as wallets, explorers, and marketplaces, benefit from this hybrid approach by first attempting to retrieve on-chain metadata for immediate trustworthiness and availability. If on-chain metadata is unavailable or limited, clients can fall back on the provided metadata URI. This method offers the best balance between efficiency, flexibility, trust, and ease of use, ensuring a smooth and user-friendly NFT experience within the Casper ecosystem.

### Token Metadata Interface: CEP95MetadataEvent

[token-metadata-interface-cep95metadataevent]: #token-metadata-interface-cep95metadataevent

```rust
pub enum CEP95MetadataEvent {
    /// Emitted whenever on-chain metadata for a token is created or updated.
    ///
    /// @param token_id - The ID of the NFT whose metadata was updated
    /// @param metadata - The raw metadata (e.g., JSON bytes)
    MetadataUpdate {
        token_id: U256,
    },
}

/// Optional interface for retrieving token metadata
pub trait CEP95TokenMetadata {
    /// Returns metadata for a given token ID.
    ///
    /// @param token_id - The NFT ID.
    /// @return Option<Vec<(String, String)>> - optional metadata stored as a list of key-value pairs.
    fn token_metadata(&self, token_id: U256) -> Option<Vec<(String, String)>>;
    
    /// Returns a URI pointing to metadata for a specific token.
    ///
    /// @param token_id - The NFT ID.
    /// @return Option<String> - A URI pointing to metadata if available.
    fn token_uri(&self, token_id: U256) -> Option<String>;
}
```

### CEP-95 On-chain Metadata Schema

When providing NFT metadata (either through the off-chain URI from token_uri or directly stored on-chain via token_metadata), it's highly recommended to follow this conventional JSON schema:

```json
{
    "title": "Casper NFT Metadata",
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
        },
        "token_uri": {
          "type": "string",
          "description": "A URI pointing to additional metadata stored off-chain (typically JSON format compatible with this schema)."
        }
    },
    "required": []
}
```

## Event Structure

[event-structure]: #event-structure

### CEP95Event Enum

Efficient event tracking is essential for indexing and off-chain applications. Your NFT standard emits events for transfers, approvals, and operator approvals.

```rust
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

- `Mint` is emitted when an NFT is minted.
- `Transfer` is emitted when an NFT is transferred.
- `Burn` is emitted when an NFT is burned.
- `Approval` is emitted when a single NFT is approved to be transferred by another account/contract.
- `RevokeApproval` is emitted when a specific NFT approval is revoked from an account/contract.
- `ApprovalForAll` is emitted when an operator’s permissions for all NFTs are enabled.
- `RevokeApprovalForAll` is emitted when an operator’s permissions for all NFTs are revoked.

### CEP95MetadataEvent Enum

```rust
pub enum CEP95MetadataEvent {
    /// Emitted whenever on-chain metadata for a token is created or updated.
    ///
    /// @param token_id - The ID of the NFT whose metadata was updated
    /// @param metadata - The metadata represented in key/value pairs
    MetadataUpdate {
        token_id: U256,
    },
}
```

- `MetadataUpdate` is emitted when an on-chain metadata for a token is created or updated.

## Usage and Implementation

[usage-and-implementation]: #usage-and-implementation

### Minting an NFT

- **Minting Process**: When minting a new NFT, the smart contract should emit a `Mint` event with `to: Some(account_hash)`. 
- **State Updates**: The contract will record the new NFT’s `token_id` with its owner and (optionally) associate metadata (via a dictionary or another persistent storage mechanism).

### Transferring an NFT

- **Safe vs. Unsafe Transfers**: Use `safe_transfer_from` when transferring to a contract (to allow the contract to confirm it can handle NFTs) and `transfer_from` for basic transfers.
- **Event Emission**: Both methods should update the owner mapping and emit a `Transfer` event.

### Approval Mechanisms

- **Per-token Approval**: The `approve` function enables an account or contract to transfer a specific NFT. If needed, revoke_approval clears that approval.
- **Operator Approvals**: The `approve_for_all` and `revoke_approval_for_all` functions allow a user to delegate transfer rights over their entire collection, with an event emitted to signal the change.


## Backend Indexing and NFT Metadata Tracking

[backend-indexing-and-nft-metadata-tracking]: #backend-indexing-and-nft-metadata-tracking

### Tracking NFT Events

Backend indexers or off-chain processors should listen for NFT-related events (`CEP95Event` event enum):

- **Minting**: Detect a `Mint` event. Extract the `token_id` and `to` address.
- **Transfers and Approvals**: Capture changes in ownership or approvals by parsing `Transfer`, `Approval`, `RevokeApproval`, `ApprovalForAll` and `RevokeApprovalForAll` events.
- **Burning**: Detect a `Burn` event. Extract the `token_id` and `from` address.

### Tracking optional NFT Onchain Metadata

Backend indexers or off-chain processors should listen for NFT Metadata related events (`CEP95MetadataEvent` event enum):

1. Detect `MetadataUpdate` event.
2. Extract the `token_id` and `metadata` from the event.
3. Store the NFT’s `token_id` and `metadata` in your database for efficient querying and display in applications

### Tracking optional NFT Offchain Metadata

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

- v1.0: Initial draft including CEP-95 and CEP95Metadata interfaces, event structures, and backend indexing guidelines.
- v1.1: Updated approach with optional Token Metadata. Added examples for tracking offchain and onchain metadata. Added recommendations for storing contract metadata. Added ref with recommendation to utilise CEP-96 for contract metadata.

### References

- [ERC-721 Standard on Ethereum](https://eips.ethereum.org/EIPS/eip-721)
- [Casper Documentation (for contract APIs and types)](https://docs.casper.network/concepts/smart-contracts)
- [Example proposals and CEPs from the Casper community](https://github.com/make-software/casper-ceps/tree/master/text)
