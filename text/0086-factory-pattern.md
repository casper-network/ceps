# Factory Pattern

## Summary

[summary]: #summary

CEP PR: [casper-network/ceps#0086](https://github.com/casper-network/ceps/pull/86)

Implementing host-side support for a factory pattern.

## Motivation

[motivation]: #motivation

This proposal introduces a host-side factory pattern to enhance the contract deployment process.

Contract developers have expressed a desire to use the factory pattern, enabling scenarios like creating multiple contracts with a custom logic. While the EE technically supports this pattern, it lacks proper tests and documentation. The existing support for this was considered an undocumented feature that relies on implementation details.

The proposed solution involves implementing a proper support on the host, adding relevant tests for deploying a factory contract and creating new contracts through it, using a newly introduced support in the smart contract API.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The host-side factory pattern introduces a way for contract developers to deploy contracts with custom logic through factory contracts. These factory contracts are created using a new entry point type called `EntryPointType::Install`, which marks an entry point as a factory method. Additionally, a new variant, `EntryPointType::Normal`, serves as the default entry point type and will behave like the existing `EntryPointType::Contract` in version 1.x. The proposal also introduces `EntryPointAccess::Abstract`, marking an entry point export as existing in the bytecode but not callable. This feature allows referencing WebAssembly exports from within entry points marked as `EntryPointType::Install`. Developers can even define nested factory contracts for more complex scenarios.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

In technical detail, this proposal introduces two new entry point types: `EntryPointType::Install` and `EntryPointType::Normal`. And a new entry point access type `EntryPointAccess::Abstract`.

* `EntryPointType::Install` marks an entry point as a factory method responsible for deploying contracts.
* `EntryPointType::Normal` behaves similarly to `EntryPointType::Contract` in version 1.x, while also serving as the default entry point type in version 2.0.
* Additionally, `EntryPointAccess::Abstract` is introduced to signify an entry point export that exists in bytecode but is not callable.
* This feature facilitates referencing contracts stored as `EntryPointType::Install`.

When creating new smart contract with an `Install` entry points, the host, then ensures that no `Abstract` entry points can be called. Entry points marked as `Install`, however, can reference the exports at the byte level, including `Install`.

Example smart contract that demonstrates new functionality is located here: https://github.com/mpapierski/casper-node/blob/gh-2064-factory-pattern/smart_contracts/contracts/test/counter-factory/src/main.rs


## Drawbacks

[drawbacks]: #drawbacks

The only known drawback of this new feature is that it does not allow deploying custom bytecode in the factory entrypoints - that means, you can't load a bytecode into the Wasm, modify it, and create new modified smart contract. All the smart contracts created with a factory pattern are sharing the same bytecode that originates from within the original session bytes.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The proposed design is chosen to enhance contract deployment while maintaining compatibility with existing entry point types.

This proposal is strictly considered as making an previously undocumented feature an officially supported pattern by making certain behaviours of the host explicit.

## Prior art

[prior-art]: #prior-art

The factory pattern is a widely recognized software design concept used in various programming contexts.

Although other blockchain platforms might have similar deployment patterns, this proposal's focus is on seamless integration with existing entry point on the Casper blockchain that sets it apart.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

Proposed names of symbols introduced in this feature may be discussed and improved through PR comments.

## Future possibilities

[future-possibilities]: #future-possibilities

This proposal sets the foundation for more advanced contract deployment patterns, potentially enabling smart contract factories that can create contracts by passing a bytecode at runtime.

Future enhancements may also involve refining the entry point types and expanding factory contract.
