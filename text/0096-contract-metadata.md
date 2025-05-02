# Casper On-Chain Contract Metadata Standard

## Summary

[summary]: #summary

This standard proposes a unified way for Casper smart contracts to self-describe with on-chain metadata, improving contract discoverability and trust. It defines optional metadata fields – `name`, `description`, `icon_uri`, and `project_uri` – which a contract can expose via dedicated entry points. These fields allow a contract to broadcast its human-readable name and purpose, along with links to an icon image and project website.

By adopting this standard, Casper developers enable wallets, network explorers, and indexers to fetch and display contract information directly from the blockchain, much like how token standards (e.g. CEP-18 for fungible tokens or CEP-95 for NFTs) provide on-chain names and symbols, underscoring the community’s move toward rich on-chain identification.

**Metadata Fields**:

- `contract_name`: A human-readable name for the contract (e.g. "Casper DEX").
- `contract_description`: A brief description of the contract’s purpose or functionality.
- `contract_icon_uri`: A URI (URL) pointing to an icon image for the contract (e.g. a logo).
- `contract_project_uri`: A URI pointing to the contract’s project website or documentation.

Each field is optional – contracts may provide all, some, or none of these. If a field is not applicable, it simply returns `None`. By keeping these fields on-chain and unchangeable after deployment, users and tools can rely on the metadata as the authentic self-declared identity of the contract, improving transparency and reducing reliance on off-chain sources.

## Motivation

[motivation]: #motivation

- Establish a uniform and simple approach for contract metadata storage. 
- Enhance contract discoverability and transparency for end-users. 
- Simplify indexing and display logic for tools like wallets and explorers.

## Specification

[specification]: #specification

Contracts implementing this standard must expose the following entry points:

```rust
pub trait CEP96ContractMetadata {
    /// Contract's human-readable name.
    fn contract_name(&self) -> Option<String>;

    /// Brief description of the contract.
    fn contract_description(&self) -> Option<String>;

    /// URI pointing to the contract's icon image.
    fn contract_icon_uri(&self) -> Option<String>;

    /// URI pointing to the project's website or documentation.
    fn contract_project_uri(&self) -> Option<String>;
}
```

Each method returns `Option<String>`, allowing contracts to omit any metadata field.


## Storage Specification

[storage-specification]: #storage-specification

To enforce simplicity, we recommend storing each metadata field under a dedicated named URef key in the contract state. Each metadata entry (if provided at installation) is stored once and cannot be altered afterward.

**URef keys**:
- `contract_name`
- `contract_description`
- `contract_icon_uri`
- `contract_project_uri`

If a field is omitted, its named `URef` is simply not created.

Here are the benefits of **URef-based Metadata Storage**:

- **Simplicity**: Easy for developers and tools to implement; minimal complexity.
- **Discoverability**: Named keys are easy to query directly via RPC.
- **Efficiency**: No need to parse or manage additional storage structures; direct access to metadata.

## Example Implementation

[example-implementation]: #example-implementation

Here's a minimal Rust implementation demonstrating this storage pattern:

```rust
use casper_contract::contract_api::{runtime, storage};
use casper_types::{CLValue, URef};

const NAME_KEY: &str = "contract_name";
const DESCRIPTION_KEY: &str = "contract_description";
const ICON_URI_KEY: &str = "contract_icon_uri";
const PROJECT_URI_KEY: &str = "contract_project_uri";

#[no_mangle]
pub extern "C" fn call() {
    // Fetch optional metadata at deployment
    let name: Option<String> = runtime::get_named_arg("contract_name");
    let description: Option<String> = runtime::get_named_arg("contract_description");
    let icon_uri: Option<String> = runtime::get_named_arg("contract_icon_uri");
    let project_uri: Option<String> = runtime::get_named_arg("contract_project_uri");

    if let Some(name) = name {
        let uref = storage::new_uref(name);
        runtime::put_key(NAME_KEY, uref.into());
    }

    if let Some(description) = description {
        let uref = storage::new_uref(description);
        runtime::put_key(DESCRIPTION_KEY, uref.into());
    }

    if let Some(icon_uri) = icon_uri {
        let uref = storage::new_uref(icon_uri);
        runtime::put_key(ICON_URI_KEY, uref.into());
    }

    if let Some(project_uri) = project_uri {
        let uref = storage::new_uref(project_uri);
        runtime::put_key(PROJECT_URI_KEY, uref.into());
    }
}

#[no_mangle]
pub extern "C" fn contract_name() {
    let value: Option<String> = read_metadata(NAME_KEY);
    runtime::ret(CLValue::from_t(value).unwrap_or_revert());
}

#[no_mangle]
pub extern "C" fn contract_description() {
    let value: Option<String> = read_metadata(DESCRIPTION_KEY);
    runtime::ret(CLValue::from_t(value).unwrap_or_revert());
}

#[no_mangle]
pub extern "C" fn contract_icon_uri() {
    let value: Option<String> = read_metadata(ICON_URI_KEY);
    runtime::ret(CLValue::from_t(value).unwrap_or_revert());
}

#[no_mangle]
pub extern "C" fn contract_project_uri() {
    let value: Option<String> = read_metadata(PROJECT_URI_KEY);
    runtime::ret(CLValue::from_t(value).unwrap_or_revert());
}

// Helper function for reading metadata from named URefs
fn read_metadata(key_name: &str) -> Option<String> {
    runtime::get_key(key_name)
        .and_then(|key| key.into_uref())
        .and_then(|uref| storage::read::<String>(uref).unwrap_or(None))
}
```

**Explanation**:
- The `call` function creates and stores metadata URefs only once at contract deployment.
- Metadata retrieval functions (`name`, `description`, etc.) expose the stored data.

**Example Usage**: If a user (or another contract) calls the `name()` entry point (via a contract call), the contract will respond with either `Some("Casper DEX")` (for example) or `None` if no name was set. Similarly, calling `icon_uri()` might return `Some("https://example.com/casperdex.png")` which a UI can use to load the icon, or `None` if the contract didn’t specify an icon.

Developers can integrate this trait easily. For instance, a contract written using the Odra framework or similar could derive or implement the `CEP96ContractMetadata` trait, and internally it could manage the storage as shown. The important part is that the compiled contract exports the four functions with the correct signatures.

## Recommended Client Usage (Wallets, Explorers, Indexers)

[recommended-client-usage-wallets-explorers-indexers]: #recommended-client-usage-wallets-explorers-indexers

To adopt this standard, tools and services in the Casper ecosystem should:

- Detect compliance by checking for named keys (`contract_name`, `contract_description`, `contract_icon_uri`, `contract_project_uri`) in the contract's state querying metadata via Casper RPC (`query_global_state`).
- Display available metadata fields clearly:
  - Use `contract_name` and `contract_icon_uri` prominently (e.g., wallet contract lists, explorer pages).
  - Use `contract_description` and `contract_project_uri` in detailed views for contract transparency and user education.
- Handle missing fields, since all fields are optional, gracefully handle `None` values.
- Cache metadata permanently after initial retrieval.

## Encouraging Community Adoption

[encouraging-community-adoption]: #encouraging-community-adoption

To maximize impact, we recommend:

- Official Casper documentation encourage developers to follow this metadata standard. 
- Frameworks and templates (like Odra) adopt this pattern by default.
- Wallets, explorers, and indexing services integrate support, making metadata discoverable and meaningful for end-users.

## Conclusion

[conclusion]: #conclusion

This Casper On-Chain Contract Metadata Standard, using named URef keys, provides a simple yet powerful mechanism for contracts to publish metadata directly on-chain. By standardizing metadata storage and access, the Casper community enhances transparency, usability, and integrity throughout the ecosystem. Broad adoption by contract developers and integration by ecosystem tools will significantly improve discoverability and user trust, positioning Casper at the forefront of user-friendly blockchain ecosystems.

## Appendix

[appendix]: #appendix

### Changelog

[changelog]: #changelog

- v1.0: Initial draft including CEP96ContractMetadata interface, and backend indexing guidelines.

### References

[references]: #references

- [Casper Documentation](https://docs.casper.network/)
- [CEP-78 (Enhanced NFT Standard)](https://github.com/casper-ecosystem/cep-78-enhanced-nft)
- [ERC-721 (Ethereum NFT Standard)](https://eips.ethereum.org/EIPS/eip-721)
- [Casper RPC Methods](https://docs.casper.network/developers/json-rpc/minimal-compliance)
- [Casper Smart Contracts Guide](https://docs.casper.network/concepts/smart-contracts)
- [Odra Framework](https://github.com/odradev/odra)
