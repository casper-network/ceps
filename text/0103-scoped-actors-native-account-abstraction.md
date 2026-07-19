# Scoped Actors: Protocol-Native Account Abstraction

## Summary

[summary]: #summary

CEP PR: [casper-network/ceps#103](https://github.com/casper-network/ceps/pull/103)

This CEP extends Casper's `AddressableEntity` account model with **scoped actors**: authorized credentials carrying a permission bitmask, an optional expiry, and an optional policy gate that restricts a credential to a single manager contract. Together with new native signature schemes (P-256 / WebAuthn passkeys) and a domain-separated gas-payer authorization, this delivers the full account-abstraction feature set — custom authentication, session keys, spending policies, and gas sponsorship — as a protocol-native capability, without ERC-4337-style bundler infrastructure or EIP-7702-style code injection.

The actor and scope semantics are deliberately aligned with Ethereum's draft [EIP-8130](https://eips.ethereum.org/EIPS/eip-8130) (Account Abstraction by Account Configuration, adopted by Base, Optimism, Coinbase and WalletConnect), so that wallet tooling built for that standard can manage Casper accounts with minimal adaptation. Where EIP-8130 must simulate protocol behavior in a singleton configuration contract, Casper implements the same semantics directly in the entity record — with fixed, chainspec-defined verification costs.

## Motivation

[motivation]: #motivation

The Casper Manifest commits to Smart Accounts (Frictionless Experience pillar), Gasless Transactions, and Agent Infrastructure (Machine Economy pillar). All three reduce to one primitive: **an account must be able to authorize a credential that is weaker than itself** — a passkey that can sign but not rotate keys, a session key that can spend 5 USDC per month at one contract and nothing else, a sponsor key that can pay gas for others but never move funds, an AI agent key with a daily cap and a whitelist.

Casper's account model is already unusually close to this. `AddressableEntity` unifies accounts and contracts as first-class protocol objects; `associated_keys` plus weighted `ActionThresholds` provide native multi-signature; TTL plus transaction-hash deduplication provide replay protection without sequential nonces. What is missing is granularity:

1. **Associated keys are all-or-nothing within an action class.** A key either clears the `deploy` threshold (and can then call anything) or it cannot act at all. There is no way to say "this key may only interact with contract X" or "this key expires Friday."
2. **Only Ed25519 and secp256k1 signatures are accepted.** No passkeys (WebAuthn/P-256), which means no device-biometric signing — the single largest UX unlock for mainstream users, and table stakes for the Manifest's "she confirms with Face ID" scenario.
3. **There is no protocol notion of a gas payer distinct from the session initiator.** Gasless Phase 2 (the `gas_payer` field and `PricingMode::Prepaid`) needs a validation rule for who may authorize payment on whose behalf.

Meanwhile the EVM ecosystem is converging on exactly this shape. EIP-8130 defines actors, scopes, policies, expiry, payer separation and canonical authenticators — implemented via a singleton configuration contract, EIP-7702 code delegation and CREATE2 counterfactual deployment, because Ethereum EOAs cannot hold structured state. Casper entities can. Adopting compatible semantics at the protocol level yields three outcomes:

- The Manifest's Smart Accounts and Agent Infrastructure initiatives get a concrete, reviewable design.
- Casper becomes interoperable with the account-management tooling being built by Base, Optimism, Coinbase and WalletConnect — the same passkey can control an account on Base and on Casper.
- Verification costs remain fixed and chainspec-defined, extending Casper's deterministic-cost differentiation to account abstraction: no other general-purpose Layer 1 can price a passkey verification identically at 10% and 90% network load.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

### New concepts

**Actor.** An authorized credential on an entity. Today's `associated_keys` entries are actors with unrestricted scope. An actor is identified by a 32-byte `actor_id` derived from its public key material, and carries:

- an **authentication scheme** (Ed25519, secp256k1, P-256, WebAuthn — extensible to ML-DSA when Quantum Safety ships),
- a **scope bitmask** describing what contexts the actor may authorize,
- an optional **expiry** timestamp,
- an optional **policy**: a target entity ("manager") plus a 32-byte commitment.

**Scope.** A bitmask of grants. `0x00` (no bits set) means unrestricted — the admin predicate, matching today's full-weight key behavior. Defined bits:

| Bit | Name | Grants |
| --- | --- | --- |
| `0x01` | `SESSION` | May initiate transactions calling any entity |
| `0x02` | `POLICY` | May initiate transactions **only** targeting the actor's policy manager |
| `0x04` | `SELF_PAYER` | May authorize payment from this entity's main purse for this entity's own transactions |
| `0x08` | `SPONSOR_PAYER` | May authorize payment from this entity's purse for **another** entity's transactions |
| `0x10` | `KEY_MGMT` | May authorize actor configuration changes (aligns with today's `key_management` threshold) |

**Policy gate.** When `POLICY` is set, the protocol enforces exactly one rule: the transaction's target entity must equal the actor's manager. Everything richer — spend caps, rate limits, allowlists, subscription logic — lives in the manager contract, which reads the actor's commitment via a new host function and enforces application-defined rules. The protocol stays small; the policy vocabulary stays open.

**Payer authorization.** A transaction may name a `gas_payer` entity distinct from the initiator. The payer signs a domain-separated payload covering the full transaction body; the payer-side signature must resolve to an actor on the payer entity carrying `SPONSOR_PAYER`. This is the validation half of Gasless Phase 2; `PricingMode::Prepaid` receipts remain an independent third payment mode.

### Examples

**A passkey user.** Maria creates an account from her phone. Her wallet registers a WebAuthn actor with scope `0x00` — her Face ID is a full-authority owner key. No seed phrase exists. Later she adds a recovery actor (her hardware key, scope `0x00`) and downgrades nothing. Signature verification costs the same fixed amount every time, regardless of network load.

**An AI agent key.** A procurement agent gets an actor with scope `POLICY | SELF_PAYER`, expiry 30 days out, manager = a spend-limit contract, commitment = hash of `{cap: 100 USDC/day, targets: [logistics API settlement contract]}`. The agent can call its manager, which validates each payment against the installed parameters and executes the x402 settlement. It cannot rotate keys, cannot touch other assets, cannot outlive its expiry. The account owner adjusts the cap from her phone by re-authorizing with a new commitment.

**A sponsored onboarding flow.** A dApp wants first-time users to transact with zero CSPR. Its sponsorship entity holds an actor scoped `SPONSOR_PAYER` used by an automated signer. User transactions name the dApp entity as `gas_payer`; the sponsor's signature authorizes payment; the user's passkey authorizes the session. Neither signature can be replayed in the other role because the payloads are domain-separated.

**Existing accounts.** Nothing changes. Every existing associated key is migrated in-place as an actor with its current weight and unrestricted scope. Weighted thresholds continue to function unchanged — this proposal is additive to Casper multisig, not a replacement for it.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Data model

Extend the entity's associated-key storage (`AssociatedKeys` under `AddressableEntity`) from `BTreeMap<AccountHash, Weight>` to a map of actor records:

```rust
pub struct Actor {
    /// Scheme + key material discriminant. Existing AccountHash values
    /// migrate as Ed25519/Secp256k1 actors losslessly.
    pub actor_id: ActorId,          // 32 bytes, scheme-tagged derivation
    pub scheme: AuthScheme,         // Ed25519 | Secp256k1 | P256 | WebAuthn | (future: MlDsa44)
    pub weight: Weight,             // preserved; participates in ActionThresholds as today
    pub scope: ScopeBits,           // u8 bitmask; 0x00 = unrestricted (admin)
    pub expiry: Option<Timestamp>,  // actor invalid when block_time > expiry
    pub policy: Option<Policy>,     // present iff scope & POLICY != 0
}

pub struct Policy {
    pub manager: EntityAddr,        // the single permitted call target
    pub commitment: [u8; 32],       // opaque, read by the manager via host function
}
```

`actor_id` derivations (EIP-8130-compatible where the scheme overlaps): Ed25519/secp256k1 actors use the existing `AccountHash`; P-256 and WebAuthn actors use `blake2b256(scheme_tag || x || y)`. Compatibility note: EIP-8130 derives P-256/passkey actorIds as `keccak256(x || y)`; wallets bridging both ecosystems map derivations at the tooling layer, since the underlying key material is identical.

Migration: at the activation upgrade, every existing associated key becomes an `Actor` with its current weight, `scope = 0x00`, `expiry = None`, `policy = None`. No action is required from any account holder. The global state migration walks entity records lazily on first write, following the established Casper 2.0 migration pattern.

### Signature verification

P-256 (FIPS 186-4, the WebAuthn default curve) and WebAuthn envelope verification are implemented as **native host verification** in the execution engine, exactly as Ed25519 and secp256k1 are today — not as callable contracts. WebAuthn verification additionally checks the authenticator-data envelope (challenge binding to the transaction hash, RP ID hash, user-presence flag) per the W3C WebAuthn Level 2 specification.

Costs are fixed in the chainspec:

```toml
[core.auth_costs]
ed25519_verification = ...      # existing
secp256k1_verification = ...    # existing
p256_verification = ...         # new, fixed
webauthn_verification = ...     # new, fixed (p256 + envelope parsing)
```

This is the deterministic-cost analog of EIP-8130's "enshrined authenticator" option — but on Casper it is the only mode, so there is no canonical-set governance problem, no allowlist divergence risk, and no STATICCALL metering. New schemes (ML-DSA-44 for Quantum Safety) are added by extending the enum and chainspec, which is precisely the cryptographic-extensibility path the protocol was designed for.

### Transaction changes

`TransactionV1` approvals generalize from `(PublicKey, Signature)` to:

```rust
pub struct ApprovalV2 {
    pub scheme: AuthScheme,
    pub public_key_material: Bytes,   // scheme-defined encoding
    pub signature: Bytes,             // scheme-defined; WebAuthn includes envelope
}
```

Two new optional header fields:

- `gas_payer: Option<EntityAddr>` — when present, gas is drawn from this entity's purse instead of the initiator's.
- `payer_approvals: Vec<ApprovalV2>` — signatures over the **payer payload**: `blake2b256(PAYER_DOMAIN_TAG || transaction_body_hash || gas_payer || initiator_addr)`. The initiator's own approvals sign the standard payload with a distinct domain tag. Binding `initiator_addr` into the payer payload prevents cross-initiator replay of payer signatures; binding `gas_payer` into the body hash signed by the initiator prevents payer substitution.

Replay protection is unchanged: TTL + transaction-hash deduplication already provide the semantics EIP-8130 constructs via its nonce-free mode (expiry + replay-identifier ring buffer). No nonce machinery is introduced. Ordered per-actor channels are deferred to Future Possibilities.

### Validation pipeline

At transaction acceptance and re-checked at execution:

1. **Resolve** each approval to an actor on the initiating entity (payer approvals resolve against the `gas_payer` entity). Unknown actor → reject.
2. **Verify** the signature under the actor's scheme at its fixed chainspec cost.
3. **Expiry check**: reject if `block_time > expiry`.
4. **Scope check** by context:
   - Session authorization requires `scope == 0x00` or `SESSION` or `POLICY`.
   - When any authorizing actor carries `POLICY` (and is not admin), the transaction's target **must** equal that actor's `policy.manager`, and the transaction must be a single-target entity call (native transfers and other targets are rejected).
   - Payment authorization: if `gas_payer` is absent, initiator-side actors must satisfy `scope == 0x00 || SELF_PAYER`. If present, payer-side actors must satisfy `scope == 0x00 || SPONSOR_PAYER`.
   - Actor-configuration changes (add/modify/remove actor, threshold changes) require `scope == 0x00 || KEY_MGMT` **and** must clear the existing `key_management` action threshold by weight.
5. **Weight check**: cumulative weight of the resolved actors must clear the relevant `ActionThreshold`, exactly as today. Scope narrows *what* a set of keys may do; weight continues to govern *how many* keys must agree.

Rule 5 is the deliberate divergence from EIP-8130, which has no native multisig. Casper composes scopes **with** thresholds: a treasury can require two `SESSION`-scoped keys totaling weight ≥ threshold, something EIP-8130 accounts must build in contract code.

### Host functions

```
casper_get_actor(entity_addr, actor_id) -> Option<(scope, expiry, policy_manager, commitment)>
```

Read-only; used by policy manager contracts to load the calling actor's commitment, and by wallet tooling. Actor configuration mutations reuse the existing key-management entry points (`add_associated_key` family), extended with scope/expiry/policy parameters and gated per rule 4.

### Interaction with other Manifest initiatives

- **Gasless Transactions (Phase 2):** `gas_payer` + `SPONSOR_PAYER` is that design. `PricingMode::Prepaid` is unaffected and composes (a prepaid receipt is a third payment mode requiring no payer signature at transaction time).
- **X402 / Agent Infrastructure:** the agent-key example above is the normative pattern; the x402 facilitator becomes a natural policy manager.
- **Quantum Safety:** ML-DSA-44 becomes one more `AuthScheme` variant; hybrid accounts hold classical and PQ actors side by side; migration is a key-management action, not an account migration.
- **EVM Execution Engine:** the EVM runtime exposes actor state through a precompile mirroring EIP-8130's `IAccountConfiguration` read surface, so Solidity contracts and EVM wallet tooling observe compatible semantics against the same underlying entity records.

## Drawbacks

[drawbacks]: #drawbacks

- **Consensus-critical surface area.** New signature schemes and a scope-enforcement pipeline in the validation path are high-blast-radius code requiring an external cryptographic audit before activation.
- **WebAuthn envelope complexity.** Authenticator-data parsing has more edge cases (flags, extensions, counter semantics) than raw curve verification; incorrect envelope validation is an authentication bypass.
- **Larger entity records.** Scope/expiry/policy per actor grows global state; policy adds two words per gated actor. Bounded by the existing associated-key count limit.
- **Wallet/SDK lift.** Every SDK, the wallet, and cspr.live must learn `ApprovalV2`, actor management and payer flows. This cost is shared with — and largely subsumed by — the Manifest's Smart Accounts commitment.
- **EIP-8130 is a moving draft.** Semantic alignment is a compatibility bet; if the EIP changes materially before finalization (Base's Cobalt deployment notwithstanding), tooling-layer mappings absorb the drift, but derivation or scope-bit changes could require chasing.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

**Why protocol-native rather than contract-based (ERC-4337 style)?** Contract-based AA exists because Ethereum could not change its account model. It costs a parallel mempool, bundler infrastructure, paymaster reputation systems, and per-operation contract simulation. Casper's account model is protocol state; extending it directly is less total machinery, uniformly cheaper, and consistent with the Manifest's claim that smart accounts are "an activation rather than a migration."

**Why align with EIP-8130 semantics instead of designing freely?** The actor/scope/policy decomposition is independently good: it keeps the protocol's enforcement minimal (one target-equality check) while pushing expressive policy into contracts. Alignment additionally buys wallet-stack compatibility with the largest AA deployment effort in the EVM world. Divergence is confined to places where Casper is strictly stronger (weighted thresholds, deterministic costs, TTL replay protection).

**Why not adopt EIP-8130's wire format and configuration-contract interface verbatim?** The wire format is RLP/EVM-shaped and presumes EIP-7702 delegation indicators, CREATE2 counterfactuals, and a 2D nonce lattice — all solutions to EOA limitations Casper does not have. Importing them would add machinery precisely where Casper's model makes it unnecessary. Semantics travel; encodings do not.

**Why keep weighted thresholds instead of adopting single-actor authorization?** Native multisig is one of Casper's oldest institutional advantages and is in production use. Scopes and weights answer orthogonal questions (what vs. how many); composing them is strictly more expressive than either alone.

**Alternative considered: authenticator contracts (user-deployable verification logic).** Rejected for the base protocol: arbitrary verification code in the acceptance path reintroduces the metering and DoS problems EIP-8130 itself works to avoid via allowlists. Casper's enum-plus-chainspec approach delivers the canonical-set outcome without an allowlist governance process. Nothing precludes contract-level ERC-1271-style verification within execution.

**Impact of not doing this:** Smart Accounts and Agent Infrastructure ship without a protocol design, or as contract-layer workarounds that forfeit the deterministic-cost and native-integration advantages the Manifest markets; passkey UX remains impossible; Gasless Phase 2 lacks its authorization model.

## Prior art

[prior-art]: #prior-art

- **EIP-8130 (draft, 2025)** — the semantic reference for actors, scopes, policies, expiry and payer separation; adopted by Base (Cobalt upgrade), Optimism, Coinbase, WalletConnect. Implemented via singleton contract + EIP-7702 because EOAs cannot carry structured state.
- **ERC-4337 (2023)** — contract-based AA via alternative mempool and bundlers; demonstrated demand (millions of smart accounts) and the operational cost of doing AA off-protocol.
- **EIP-7702 (Pectra, 2025)** — EOA code delegation; the underlying mechanism 8130 builds on, and the clearest evidence that Ethereum is retrofitting toward a unified account/contract model Casper adopted in 2.0.
- **EIP-7701** — fully general native AA with new opcodes; maximal flexibility at the cost of the arbitrary-validation-code problem.
- **Casper CEP-0057 / existing account model** — weighted associated keys and action thresholds; this proposal generalizes rather than replaces them.
- **W3C WebAuthn Level 2** — the envelope verification standard for passkey actors.
- **casper-eip-712 / CEP permit extensions (2612, 3009)** — the typed-signing groundwork already used for Gasless Phase 1 and x402.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

1. **Scope-bit numbering freeze.** Should bit values track EIP-8130's assignments exactly (including its `NONCE` bit, meaningless on Casper) for tooling symmetry, or renumber cleanly as specified here? Resolve before implementation; renumbering later is a hard fork of wallet assumptions.
2. **Policy gate vs. weighted co-signing.** When a `POLICY` actor co-signs with an admin actor in one approval set, does the policy gate bind (strictest-wins) or does admin authority dominate? Proposed: strictest-wins — any non-admin `POLICY` approval binds the gate. Needs security review.
3. **WebAuthn envelope profile.** Which authenticator-data extensions are accepted, and is the signature counter enforced (privacy/replay trade-off)? Requires a companion specification.
4. **Actor count and record-size limits.** Chainspec ceilings for actors per entity and policy storage growth.
5. **`actor_id` derivation compatibility.** Adopt keccak-based derivations for P-256/WebAuthn actors to match EIP-8130 byte-for-byte, or keep blake2b and map at tooling level? Cross-ecosystem passkey portability argues for the former; hash-function consistency within Casper argues for the latter.
6. **Fee treatment of failed payer-authorized transactions.** Who bears the cost of a transaction whose session validation fails after payer signature verification succeeded? Must be resolved to prevent sponsor griefing.

## Future possibilities

[future-possibilities]: #future-possibilities

- **Ordered per-actor nonce channels** for high-throughput automation keys that need strict sequencing beyond TTL semantics.
- **Delegate actors** (account B authorized as an actor on account A, EIP-8130's delegate authenticator): corporate authority chains without key sharing. Deliberately deferred; depth-1 delegation semantics deserve their own CEP.
- **Native session receipts for x402**: policy managers issuing standing payment authorizations consumed by facilitators, merging this CEP's agent keys with `PricingMode::Prepaid`.
- **Post-quantum actors** (ML-DSA-44) as a scheme addition under Quantum Safety — hybrid classical/PQ accounts become a pure configuration matter.
- **Account lock / rotation notice periods** (EIP-8130's lock semantics) for custody-grade change control on institutional accounts.
- **Protocol-level compliance hooks**: Native Token Registry transfer hooks reading actor scopes, enabling "only accredited-investor-scoped actors may receive this token" — connecting this CEP to the Compliant Security Tokens initiative.
- **EVM-side full parity**: exposing actor management (not just reads) through the EIP-8130 precompile surface once the EVM engine ships, making Casper the first non-EVM L1 where 8130 tooling works unmodified.
