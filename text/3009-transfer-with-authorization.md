# Casper Payments: Transfer with Authorization (CEP-3009)

## Summary
A standard interface for off-chain authorization of CEP-18 token transfers
via signed messages. The standard is inspired by the ERC-3009 standard
from Ethereum and is adapted to Casper's native multi-scheme signatures
(ed25519, secp256k1). A token holder signs a typed message authorizing a
specific transfer (recipient, amount, validity window, unique nonce).
Depending on the entry point, that signed authorization may then be
submitted either by any relayer or only by the intended recipient —
enabling gas-less transfers, recipient-pulled transfers, and
payment-style flows.

This CEP extends [CEP-18](0018-token-standard.md). It does not change
CEP-18's `transfer` / `transfer_from` semantics; it adds new entry points
that produce the same balance state change as `transfer`, with relay
rules defined per entry point.

In contrast to [CEP-2612](2612-permit-extension.md), which signs an
allowance for later pull-style spending, CEP-3009 signs a one-shot
transfer that takes effect when relayed.

## Prior art
The main source of influence is [EIP-3009](https://eips.ethereum.org/EIPS/eip-3009).

The signed payload follows [EIP-712](https://eips.ethereum.org/EIPS/eip-712)
typed data hashing. EIP-712 is treated here as a dependency rather than
re-specified — implementations are expected to use [EIP-712 toolkit for
Casper](https://github.com/casper-ecosystem/casper-eip-712).

## Specification

The CEP-3009 extension is defined by:
- three new mutating entry points (`transfer_with_authorization`,
  `receive_with_authorization`, `cancel_authorization`),
- one read entry point (`authorization_state`),
- the EIP-712 typed data definitions for the three structs above
  and the domain separator,
- events,
- error codes,
- storage structure for replay-protection nonces and the EIP-712 chain
  identifier.

Below definitions use Rust syntax, but they are not Rust specific. `Bytes`
denotes the Casper `bytesrepr::Bytes` type; nonces are exactly 32 bytes.

### Entry point interface

Contracts implementing this standard must expose the following entry
points in addition to the CEP-18 interface:

```rust
pub trait CEP3009Interface {
    /// Returns true if `(authorizer, nonce)` has already been used or
    /// canceled. False otherwise. Wallets should pick a fresh nonce for
    /// which this returns false before signing.
    fn authorization_state(&self, authorizer: Key, nonce: Bytes) -> bool;

    /// Executes a `from` → `to` transfer of `value` tokens, given
    /// `from`'s signed authorization.
    ///
    /// - The signature is verified against the `TransferWithAuthorization`
    ///   typed payload (see below).
    /// - The transaction caller (`env().caller()`) is irrelevant — this
    ///   entry point is designed to be relayed by any third party.
    ///
    /// On success:
    /// - the transfer is executed with the same balance effect as `from`
    ///   calling CEP-18's `transfer(to, value)`,
    /// - `(from, nonce)` is recorded as used,
    /// - an `AuthorizationUsed { authorizer: from, nonce }` event is
    ///   emitted.
    ///
    /// The call must revert:
    /// - with `NonceAlreadyUsed` if `(from, nonce)` is already used or
    ///   canceled,
    /// - with `AuthorizationNotYetValid` if the current block time is
    ///   not strictly greater than `valid_after`,
    /// - with `AuthorizationExpired` if the current block time is not
    ///   strictly less than `valid_before`,
    /// - with `InvalidPublicKey` if `from` is not the `Address` derived
    ///   from `public_key`,
    /// - with `InvalidSignature` if `signature` does not verify against
    ///   `public_key` over the recomputed digest,
    /// - with the appropriate CEP-18 error if the underlying transfer
    ///   fails (e.g. `InsufficientBalance`).
    fn transfer_with_authorization(
        &mut self,
        from: Key,
        to: Key,
        value: U256,
        valid_after: u64,
        valid_before: u64,
        nonce: Bytes,
        public_key: PublicKey,
        signature: Bytes
    );

    /// Identical to `transfer_with_authorization` except that the
    /// transaction caller MUST equal `to`. This binds the relayer to the
    /// intended recipient — useful for receive-style flows where the
    /// recipient pulls funds and the authorization should not be
    /// front-runnable by an arbitrary relayer.
    ///
    /// The signature is verified against the `ReceiveWithAuthorization`
    /// typed payload — a distinct typehash from
    /// `TransferWithAuthorization`, so a signature for one cannot be
    /// replayed against the other.
    ///
    /// In addition to the reverts of `transfer_with_authorization`, this
    /// call must revert:
    /// - with `InvalidCaller` if `env().caller() != to`.
    fn receive_with_authorization(
        &mut self,
        from: Key,
        to: Key,
        value: U256,
        valid_after: u64,
        valid_before: u64,
        nonce: Bytes,
        public_key: PublicKey,
        signature: Bytes
    );

    /// Cancels an unused authorization. After cancellation, any
    /// subsequent `transfer_with_authorization` or
    /// `receive_with_authorization` call using `(authorizer, nonce)`
    /// reverts with `NonceAlreadyUsed`.
    ///
    /// Cancellation does NOT depend on `valid_after` / `valid_before` —
    /// an expired authorization may still be canceled to ensure it can
    /// never be used.
    ///
    /// The signature is verified against the `CancelAuthorization` typed
    /// payload.
    ///
    /// On success: `(authorizer, nonce)` is recorded as used (the same
    /// storage entry as a consumed authorization), and an
    /// `AuthorizationCanceled { authorizer, nonce }` event is emitted.
    ///
    /// The call must revert:
    /// - with `AuthorizationUsed` if `(authorizer, nonce)` is already
    ///   used or canceled,
    /// - with `InvalidPublicKey` if `authorizer` is not the `Address`
    ///   derived from `public_key`,
    /// - with `InvalidSignature` if `signature` does not verify against
    ///   `public_key` over the recomputed digest.
    fn cancel_authorization(
        &mut self,
        authorizer: Key,
        nonce: Bytes,
        public_key: PublicKey,
        signature: Bytes
    );
}
```

#### Signature scheme — divergence from EIP-3009

EIP-3009 passes signatures as `(v, r, s)` because Ethereum recovers the
signer's address from the signature itself via ECDSA recovery and then
asserts that the recovered address equals the signed `from` /
`authorizer`. Casper supports multiple signature schemes (ed25519 and
secp256k1) and does not recover the signer from the signature, so this
CEP takes an explicit `(public_key, signature)` pair instead.

To preserve the property that an authorization may only authorize its
own signer's account, every entry point that accepts a signature MUST
enforce both:

1. The signed-payload subject (`from` for transfer/receive,
   `authorizer` for cancel) equals `Address::from(public_key)`, AND
2. `signature` is a valid signature of `public_key` over the EIP-712
   digest of the corresponding typed data.

Without (1), a third party could sign a digest declaring an unrelated
subject with their own key and have the contract act against that
subject's storage (e.g. permanently canceling another holder's freshly
issued authorization). Failures of (1) MUST revert with
`InvalidPublicKey`; failures of (2) MUST revert with `InvalidSignature`.

### Validity window

`valid_after` and `valid_before` are unix timestamps in seconds.

An authorization is valid when:

```
valid_after < now_secs < valid_before
```

Both bounds are strict. Conventional sentinel values are `0` for
`valid_after` (always valid from the start) and `u64::MAX` for
`valid_before` (never expires).

### Signed payloads (EIP-712)

CEP-3009 defines three EIP-712 typed structs. For each, the digest is
computed using the standard EIP-712 rule
(`keccak256("\x19\x01" || domainSeparator || keccak256(typeHash || encodedData))`).

The `domainSeparator` is defined as:
- `name` — the CEP-18 token's `name`,
- `version` - the current major version of the signing domain,
- `chain_name` — the contract's `chain_name` (see Storage below); It should be 
   in the [CAIP-2](https://github.com/ChainAgnostic/namespaces/blob/main/casper/caip2.md)
   format,
- `contract_package_hash` — the contract's own address.

Using `EIP-712` notation, the domain separator type string is:

```
EIP712Domain(string name,string version,string chain_name,bytes32 contract_package_hash)
```

#### TransferWithAuthorization

Type string:

```
TransferWithAuthorization(address from,address to,uint256 value,uint256 validAfter,uint256 validBefore,bytes32 nonce)
```

Typehash:

```
TRANSFER_WITH_AUTHORIZATION_TYPEHASH = 0x7c7c6cdb67a18743f49ec6fa9b35f50d52ed05cbed4cc592e13b44501c1a2267
```

#### ReceiveWithAuthorization

Type string:

```
ReceiveWithAuthorization(address from,address to,uint256 value,uint256 validAfter,uint256 validBefore,bytes32 nonce)
```

Typehash:

```
RECEIVE_WITH_AUTHORIZATION_TYPEHASH = 0xd099cc98ef71107a616c4f0f941f04c322d8e254fe26b3c6668db87aae413de8
```

#### Transfer / Receive encoded message

`Address` encoding is defined as 33 bytes, where the first byte is a type tag:
- `0x00` for `AccountHash`,
- `0x01` for contract package's `Hash`.

The encodeData value is the concatenation of the following EIP-712-encoded
fields, in order:

- `from` encoded as EIP-712 `address`,
- `to` encoded as EIP-712 `address`,
- `value` encoded as EIP-712 `uint256`,
- `valid_after` encoded as a `uint256`,
- `valid_before` encoded as a `uint256`,
- `nonce` as a 32-byte value (right-padded with zeros if shorter).

#### CancelAuthorization

Type string:

```
CancelAuthorization(address authorizer,bytes32 nonce)
```

Typehash:

```
CANCEL_AUTHORIZATION_TYPEHASH = 0x158b0a9edf7a828aad02f63cd515c68ef2f50ba807396f6d12842833a1597429
```

Encoded message data:

- `authorizer` encoded as EIP-712 `address`,
- `nonce` as a 32-byte value.

### Events interface

State changes are communicated via events emitted using the
[Casper Event Standard](https://github.com/make-software/casper-event-standard).

```rust
pub enum CEP3009Event {
    /// Emitted on successful `transfer_with_authorization` or
    /// `receive_with_authorization`.
    AuthorizationUsed {
        /// The signer of the consumed authorization.
        authorizer: Key,
        /// The nonce of the consumed authorization.
        nonce: Bytes,
    },

    /// Emitted on successful `cancel_authorization`.
    AuthorizationCanceled {
        /// The signer of the canceled authorization.
        authorizer: Key,
        /// The nonce of the canceled authorization.
        nonce: Bytes,
    },
}
```

A successful transfer additionally produces the CEP-18 `Transfer` event
emitted by the underlying CEP-18 token contract being driven.

### Error codes

The CEP-3009 extension contract should revert with the following error
codes when the appropriate conditions are met:

```rust
pub enum CEP3009Error {
    /// `(authorizer, nonce)` has already been used or canceled.
    NonceAlreadyUsed = 37_000,
    /// The current block time is not strictly less than `valid_before`.
    AuthorizationExpired = 37_001,
    /// The current block time is not strictly greater than `valid_after`.
    AuthorizationNotYetValid = 37_002,
    /// `signature` does not verify against `public_key` over the
    /// recomputed digest.
    InvalidSignature = 37_003,
    /// The signed-payload subject (`from` / `authorizer`) is not the
    /// `Address` derived from `public_key`.
    InvalidPublicKey = 37_004,
    /// `env().caller() != to` in `receive_with_authorization`.
    InvalidCaller = 37_005,
    /// The `(authorizer, nonce)` pair targeted by `cancel_authorization`
    /// is already used or canceled.
    AuthorizationUsed = 37_006,
}
```

These codes are disjoint from the CEP-18 (`60_001`–`60_003`) and
CEP-2612 (`36_000`–`36_001`) ranges.

### Storage interface

Querying authorization state externally — which a wallet must do to pick
a fresh nonce — requires direct contract storage access. This CEP fixes
the on-chain layout of the two state items introduced by the extension.

#### Simple values

The chain name used in the EIP-712 domain separator is stored in the
contract's named keys:

- The chain name is stored under the key `chain_name` with the type
  `String`.
- Must use CAIP-2 chain ids (e.g.: casper:casper, casper:casper-test, or
  casper:casper-net-1).

#### Used nonces

Used and canceled nonces share a single dictionary stored under the
named key `used_nonces`. It is a key-value storage where:

- The key is a pair `(authorizer, nonce)`.
- The value is a `bool`. A value of `true` means the nonce has been
  consumed by a transfer OR canceled. A missing entry (treated as
  `false`) means the nonce is still available.

The dictionary key is created using the same algorithm as CEP-18
allowances:

- The `authorizer` and `nonce` are each converted to bytes using
  standard `CLType` encoding.
- The two byte vectors are concatenated, `authorizer` first.
- The concatenated value is hashed using the blake2b hashing algorithm.
- The hash is encoded to a string using hex encoding.

An example used-nonce key generation implementation:

```rust
fn used_nonce_key(authorizer: Key, nonce: Bytes) -> String {
    let mut preimage = Vec::new();
    preimage.extend_from_slice(&authorizer.to_bytes().unwrap());
    preimage.extend_from_slice(&nonce.to_bytes().unwrap());
    let hash_bytes = runtime::blake2b(preimage);
    hex::encode(hash_bytes)
}
```

The same value is exposed read-only through the `authorization_state`
entry point, which returns `false` when no entry is present.

#### Generating new nonces

It is user's responsibility to generate fresh nonces for new authorizations. The
best practice is to use a cryptographically secure random generator to produce a
32-byte nonce, and to check that the generated nonce is not already used.

## Existing implementations

A reference implementation of CEP-3009 is available at:
- https://github.com/odradev/odra/blob/release/2.7.0/modules/src/cep3009.rs
