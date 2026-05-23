# Casper Payments: Permit Extension (CEP-2612)

## Summary
A standard interface for off-chain approval of CEP-18 token allowances via
signed messages. The standard is inspired by the ERC-2612 standard from Ethereum
and is adapted to Casper's native multi-scheme signatures (ed25519, secp256k1).
A token holder signs a typed message authorizing a `spender` to receive an
allowance, and any party may submit that signature to the contract to set the
allowance on the holder's behalf — enabling gas-less approvals for the holder.

This CEP extends [CEP-18](0018-token-standard.md). It does not change CEP-18's
`approve` / `allowance` / `transfer_from` semantics; it adds a new entry point
that produces the same allowance state change as `approve` while being callable
by a third party.

## Prior art
The main source of influence is [EIP-2612](https://eips.ethereum.org/EIPS/eip-2612).

The signed payload follows [EIP-712](https://eips.ethereum.org/EIPS/eip-712)
typed data hashing. EIP-712 is treated here as a dependency rather than
re-specified — implementations are expected to use [EIP-712 toolkit for
Casper](https://github.com/casper-ecosystem/casper-eip-712).

## Specification

The CEP-2612 extension is defined by:
- one new entry point (`permit`),
- the EIP-712 typed data definition for the `Permit` struct and its domain separator,
- error codes,
- storage structure for replay-protection nonces and the EIP-712 chain
  identifier.

The extension reuses CEP-18's `SetAllowance` event — `permit` produces the
same allowance state change as `approve` and emits the same event. No new
events are introduced by this CEP.

Below definitions use Rust syntax, but they are not Rust specific.

### Entry point interface

Contracts implementing this standard must expose the following entry point
in addition to the CEP-18 interface:

```rust
pub trait CEP2612Interface {
    /// Sets `value` as the allowance of `spender` over `owner`'s tokens,
    /// given `owner`'s signed approval.
    ///
    /// - `owner` is the token holder whose tokens are being approved.
    /// - `spender` is the address that will be allowed to spend the tokens.
    /// - `value` is the allowance amount.
    /// - `deadline` is a block-time timestamp after which the permit is no
    ///   longer accepted, expressed in the same units as the host
    ///   environment's `get_block_time()`. The sentinel value `u64::MAX`
    ///   disables the expiry check.
    /// - `public_key` is the signer's Casper public key.
    /// - `signature` is `public_key`'s signature over the EIP-712 digest
    ///   of the `Permit` typed data.
    ///
    /// On success:
    /// - the allowance of `spender` over `owner` is set to `value` (i.e.
    ///   the same effect as `owner` calling `approve(spender, value)`),
    /// - the `owner`'s permit nonce is incremented by one,
    /// - a CEP-18 `SetAllowance { owner, spender, allowance: value }`
    ///   event is emitted.
    ///
    /// The call must revert:
    /// - with `PermitExpired` if `deadline != u64::MAX` and the current
    ///   block time is strictly greater than `deadline`,
    /// - with `InvalidPublicKey` if `owner` is not the `Address` derived
    ///   from `public_key` (see "Signature scheme" below),
    /// - with `InvalidSignature` if `signature` does not verify against
    ///   `public_key` over the recomputed digest,
    /// - with `InvalidNonce` if the `nonce` is not exactly equal to
    ///   the one expected by the contract.
    ///
    /// The transaction caller (`env().caller()`) is irrelevant — `permit`
    /// is designed to be relayed by any third party.
    fn permit(
        &mut self,
        owner: Key,
        spender: Key,
        value: U256,
        deadline: u64,
        public_key: PublicKey,
        signature: Bytes
    );
}
```

#### Signature scheme — divergence from EIP-2612

EIP-2612 passes the signature as `(v, r, s)` because Ethereum recovers the
signer's address from the signature itself via ECDSA recovery and then
asserts that the recovered address equals `owner`. Casper supports
multiple signature schemes (ed25519 and secp256k1) and does not recover
the signer from the signature, so this CEP takes an explicit
`(public_key, signature)` pair instead.

To preserve the property that a permit signature only authorizes its own
`owner`'s allowance, implementations MUST enforce:

1. `owner == Address::from(public_key)` — i.e. `owner` is the account
   address corresponding to `public_key` AND
2. `signature` is a valid signature of `public_key` over the EIP-712
   digest of the `Permit` typed data (described below).

Without (2) a third party could sign a digest containing an unrelated `owner`
field with their own key and have the contract set that `owner`'s allowance. (1)
reverts with `InvalidPublicKey`, while (2) reverts with `InvalidSignature` if
not satisfied.

### Signed payload (EIP-712)

The signed payload is the EIP-712 digest of a `Permit` typed struct.

The Permit type string is:

```
Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)
```

Its typehash is keccak256 of the type string above:

```
PERMIT_TYPEHASH = 0x6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9
```

`Address` encoding is defined as 33 bytes, where the first byte is a type tag:
- `0x00` for `AccountHash`,
- `0x01` for package's `Hash`.

The encodeData value is the concatenation of the following EIP-712-encoded
fields, in order:

- `owner` encoded as an EIP-712 `address`,
- `spender` encoded as an EIP-712 `address`,
- `value` encoded as an EIP-712 `uint256`,
- `nonce` encoded as an EIP-712 `uint256` — equal
  to the current on-chain `permit_nonces[owner]` value at the time the
  signature was produced,
- `deadline` encoded as an EIP-712 `uint256` (32 bytes, big-endian,
  left-padded).

The final digest is computed using the standard EIP-712 rule
(`keccak256("\x19\x01" || domainSeparator || keccak256(typeHash || encodedData))`).

The `domainSeparator` is defined as:
- `name` — the CEP-18 token's `name`,
- `chainId` — the contract's `chain_name` (see Storage below); It should be 
   in the [CAIP-2](https://github.com/ChainAgnostic/namespaces/blob/main/casper/caip2.md)
   format,
- `verifyingContract` — the contract's own address.

The salt and version fields are not used.

### Error codes

The CEP-2612 extension contract should revert with the following error
codes when the appropriate conditions are met:

```rust
pub enum CEP2612Error {
    /// Either `signature` does not verify against `public_key` over the
    /// recomputed digest.
    InvalidSignature = 36_000,
    /// The current block time is strictly greater than `deadline`, and
    /// `deadline` is not the sentinel `u64::MAX`.
    PermitExpired = 36_001,
    /// The `nonce` is not exactly equal to the one expected by the contract.
    InvalidNonce = 36_002,
    /// `owner` is not the `Address` derived from `public_key`.
    InvalidPublicKey = 36_003,
}
```

These codes are disjoint from the CEP-18 error code range
(`60_001`–`60_003`).

### Storage interface

Querying the permit nonce externally — which a wallet must do before
producing a signature — requires direct contract storage access. This
CEP fixes the on-chain layout of the two state items introduced by the
extension.

#### Simple values

The chain name used in the EIP-712 domain separator is stored in the
contract's named keys:

- The chain name is stored under the key `chain_name` with the type
  `String`.

#### Permit nonces

Permit nonces are stored in the dictionary under the named key
`permit_nonces`. It is a key-value storage where:

- The key is the account address of the token holder (`owner`).
- The value is the current nonce for that holder, as a `U256`. Holders
  that have never used `permit` have no stored value; their nonce is
  considered to be `0`.

The key is created using the same algorithm as CEP-18 balance keys:

- The account address is converted to bytes using standard `CLType`
  encoding.
- The bytes are encoded to string using base64 encoding.

An example permit-nonce key generation implementation:

```rust
use base64::prelude::{Engine, BASE64_STANDARD};

fn permit_nonce_key(account: Key) -> String {
    let preimage = account.to_bytes().unwrap();
    BASE64_STANDARD.encode(preimage)
}
```

Each successful `permit` call increments the stored nonce by one. The
nonce that must appear in the signed `Permit` struct is the value
present at the moment of signing (typically read off-chain immediately
before signing).

## Existing implementations

A reference implementation of CEP-2612 is available at:
- https://github.com/odradev/odra/blob/feature/gasless-op/modules/src/erc2612.rs (TODO: update after acceptance).
