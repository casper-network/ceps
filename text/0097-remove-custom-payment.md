# Remove Custom Payment Support and Lower Minimum Transaction Cost

## Summary

[summary]: #summary

CEP PR: [casper-network/ceps#102](https://github.com/casper-network/ceps/pull/102)

The original design of the Casper blockchain expected the sender of each transaction to include a separate wasm responsible, when executed, for buying the gas that would be used to execute the primary logic of that transaction. This was the original payment mechanism, and all versions of the blockchain as of DevNet onward included an implementation to allow this. 

> **Note**: This was initially intended to be the only way to buy gas to pay for the execution of transactions, but other options soon proliferated.

The blockchain issues every account a purse to hold token in (the `main_purse`), but permits an account to have permissions on multiple purses. The wasm based payment logic was intended to allow the sender of the transaction to provide bespoke logic to pay for gas from among any of the purses available to them, and / or record any accounting details.

While this facility for custom payment logic was a theoretically useful feature, in practice everyone paid for their gas using their `main_purse`, using the so-called "Standard Payment" wasm, which did the bare minimum necessary to affect payment for the gas of the upcoming transaction execution.

Soon, still prior to mainnet, having to constantly pass the same payment wasm on every such transaction was very inconvenient. This led to the request to interpret a lack of payment wasm as instruction for the node to use the Standard Payment wasm by default, which was implemented. 

Still later, this default behavior was uplifted further into native logic and the Standard Payment wasm was dispensed with entirely. 

The option for a sender to provide Custom Payment wasm was retained for retro-compatibility, but saw little usage. 

This CEP proposes two related changes: (1) the removal of Custom Payment support, and (2) the consequent lowering of the minimum transaction cost (the effective minimum balance / minimum transfer amount) that was historically anchored to the 2.5 CSPR failed-payment penalty. The second change is enabled by the first: removing Custom Payment removes the mechanism that made 2.5 CSPR a load-bearing constant.

### Deprecation, Removal, Restoration
At some point during the 1.5 development timeline, planning for 2.0 was ongoing in the background. The nature of Custom Payment was in the way of various planned 2.0 features, and after discussion the decision was made to deprecate the feature with removal to follow with 2.0 release.

This removal did occur, along with the removal of other deprecated functionality, and initial rel candidates for 2.0 no longer supported it.

However, prior to release there was general pushback on deprecation of features. A requirement was given to prioritize retro-compatibility across the board, and add back such removed functionality.

There were pragmatic reasons to do this, but it was not straightforward to restore Custom Payment specifically. The original payment processing had been entirely replaced for 2.0, so unlike most other deprecated features that were un-removed, bringing back Custom Payment required re-architecture and re-implementation. 

Secondarily, some advanced features intended for 2.0 release (contract self-pay) were blocked due to incompatibility and remain unrealized.

## Motivation

[motivation]: #motivation

Despite its existence and the lengths taken to retain the feature, it has never been well liked or well-used. As described in the summary, there was immediately a push towards standardizing payment and ultimately making it a native default operation. 

In practice, the Custom Payment option has gone unused, and has not justified its complexity within the system. Most people don't know or remember it is an option. Those who do generally conclude that the level of effort and oversight necessary to set up and administer various purses, and write and maintain bespoke logic to utilize them for payment, is simply not worth it. 

Furthermore, 2.0 comprehensively rearchitected the payment, fee, refund, and reward mechanisms of the blockchain, and moved indication of how to pay for execution into the transaction itself. Hooks for further payment handling options for things like contract self-pay and prepay were also built into the model. The overall stance of the 2.0 approach is to support multiple specific built-in payment options as a first level concern. Custom Payment is at cross purposes to this intent at best and in some cases is simply incompatible, and the logic reflects this schism. 

Beyond these higher order notions of lack of use, design friction, and tortured logic, the nature of Custom Payment intrinsically causes deep-seated practical challenges.

### The Payment Paradox

When Custom Payment wasm is provided, the bytes are opaque. If executed, the logic contained within that wasm may or may not do any particular thing. By convention, it is expected that after it is executed the balance of a system controlled purse that starts empty will be sufficient to cover the gas cost of the actual work of transaction. If it is, that work is allowed to process. If it isn't that work is not allowed to process.

In the happy path, such wasm is executed, it runs and halts on its own, the balance check of the payment purse passes, and work proceeds. 

There are multiple sad paths. The most prominent of which is, it would be an attack vector if a malicious actor could spam the blockchain with transactions including bogus wasm that spins indefinitely for free. 

Thus, this must be protected against in some way.

#### Payment Wasm Gas Cost

As explained, Custom Payment is externally provided opaque wasm, which the node executes without knowledge of what the logic will do. Famously, we cannot know if there is an intrinsic halting state. Thus, gas must be paid to cover the cost of executing this wasm so that we can meter and forcibly halt it if the execution exceeds the amount of gas available. In this manner, it is ensured that the payment wasm will halt if executed. 

> The payment wasm needs gas to be paid so that it can be executed. 
> 
> To pay for gas, we run the payment wasm. 
>  
> *How do you pay for the thing that pays for the thing in a way that makes sure that the thing paying for the thing is itself paid for?*
>
> Thus, the **payment paradox**, or a classic chicken-and-egg problem, if you prefer.
>

There are ways to avoid or mitigate this payment paradox. The Casper blockchain checks the transaction initiator's `main_purse` to ensure they have sufficient token balance to cover a flat penalty amount if their payment logic fails to transfer the expected payment into the system payment purse.

If there is enough to cover the penalty for failed payment, we allow the payment wasm to execute. If it errors or fails to pay the expected amount, the penalty is taken from the initiator's `main_purse`.

> **Note**: the need for a discoverable purse to charge a penalty to was the original forcing function to add an explicit `main_purse` to the `Account` model.

#### The 2.5 CSPR Failed Payment Penalty 

The penalty to charge for failed payment was pegged at ***2.5*** CSPR pre-mainnet, based upon projected expectations of value. This constant was never recalculated or changed to a dynamic amount based upon real market conditions.

Superficially, this would be an implementation detail. However, it became a center of gravity that warped adjacent considerations. 

Since an account with less than 2.5 CSPR in its `main_purse` would not be permitted to move forward in execution processing, 2.5 CSPR was effectively a minimum balance. Eventually, this implicit effective minimum was formalized into an enforced minimum.
       
Because transferring less than 2.5 CSPR to a new account would result in a zombie account not allowed to execute anything, the minimum transfer amount was set to 2.5 CSPR. 
      
In this way, the magical value of 2.5 became insinuated into various pricing considerations as a fixed value that could be reasoned about and compared to. In particular, for arbitrarily costed native functionality that required a wasm opcode equivalent cost, 2.5 CSPR is where the reasoning about a possible value would start.

Over time, the original purpose of this magical number as the custom payment penalty amount became esoteric.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

After this change, there is exactly one way to pay for a transaction: the native payment handling introduced in 2.0, where the intent to pay for execution is expressed in the transaction itself. The ability to attach opaque Custom Payment wasm to a transaction is removed.

From the perspective of a typical user or dApp developer, nothing changes — they already pay from their `main_purse` via native payment and never authored Custom Payment wasm. The change is only observable to the (effectively empty) set of senders who were attaching Custom Payment wasm; such transactions are now rejected at acceptance time rather than executed.

Because the 2.5 CSPR failed-payment penalty is what forced an effective minimum balance and minimum transfer amount, removing Custom Payment removes the reason that constant had to exist. The **minimum transaction cost** — expressed through the enforced minimum balance for execution permission and the minimum transfer amount — is correspondingly lowered. Rather than replace 2.5 CSPR with a new hard-coded magic number, this CEP proposes that these minimums be controlled by a chainspec value, so they can be tuned by network governance to reflect real market conditions instead of a pre-mainnet projection.

Concretely:
- A transaction that includes Custom Payment wasm is rejected by the node.
- The enforced minimum `main_purse` balance required to permit execution is no longer pinned to 2.5 CSPR; it becomes a chainspec-configurable value that can be set materially lower.
- The minimum transfer amount is likewise decoupled from 2.5 CSPR and driven by the same or a related chainspec value.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Removal of Custom Payment

The steps to enact the removal are simple, and the effort is low for someone familiar with the logic. 

The majority of the custom payment handling logic is at the top of the transaction processing loop in the `contract_runtime`. 
- All the relevant logic would be removed, greatly simplifying the transaction processing loop.
- All the relevant tests would also be removed.
- The remaining payment logic would be restored to the intended 2.0 form.

Secondarily, with the removal of the logic to process custom payment, nodes should reject transactions containing custom payment.
- The `transaction_acceptor` would be modified to reject such transactions.
- A new test would be added to prove rejection of a transaction with custom payment wasm attached.

### Lowering the Minimum Transaction Cost

With Custom Payment removed, the 2.5 CSPR failed-payment penalty no longer needs to exist, and the values that were anchored to it can be lowered and made configurable:

- The enforced minimum `main_purse` balance required for execution permission is moved from the hard-coded 2.5 CSPR to a chainspec value (e.g. a `minimum_balance` / minimum-execution-balance setting).
- The minimum transfer amount is decoupled from 2.5 CSPR and driven by a chainspec value (this may be the same value, or a distinct minimum-transfer setting, to be decided during implementation review).
- Existing accounts and balances are unaffected; the change only relaxes the lower bounds going forward.

The specific default value(s) for these chainspec settings are intentionally left to be fixed during implementation/governance review rather than baked into this CEP, so that the number reflects market conditions at activation rather than reintroducing a new fixed magic constant. The design intent is that these minimums be materially lower than 2.5 CSPR and adjustable without a code change.

### Historical Concerns

There are no historical concerns. All preexisting transactions, having executed in the past, would not be affected. The `block_synchronizer` would continue to acquire such transactions for historical blocks when necessary. 

Prior to version 1.5 of the network, when a node joining the network acquired a historical block, they would execute it for themselves. This required the software to retain the ability to execute all historical transactions, retaining code execution paths in perpetuity and dispatching transactions into the relevant path. 

With the introduction of `fast_sync` in 1.5, nodes that need historical blocks instead acquire from peer nodes all parts of the block itself and all the side effects of that block having been executed, including a full slice of global state, and enough finality signatures from validators of that era to prove the block is legitimate. 

Therefore, the node software does not need to retain logic for removed features to allow execution of historical blocks.

## Drawbacks

[drawbacks]: #drawbacks

The Custom Payment feature has been on the chopping block for removal multiple times, and was even removed in a series of rel candidates before being re-added.  

The following have been the counter-arguments put forth to keep it:

- It is a flexible option that combines custom wasm with multiple purses. It provides some ineffable utilitarian value, despite the lack of usage or enthusiasm for it.
- Retro-compatibility, though (again) this is largely abstract due to the general lack of usage.

For the minimum transaction cost change, the primary drawback is that lowering the minimum balance / minimum transfer reduces the economic friction that discourages dust accounts and dust transfers. Making the value chainspec-controlled mitigates this by allowing the network to raise it again if abuse emerges, without requiring a code change.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- **Why remove rather than keep-and-fix?** Custom Payment's core problem — the payment paradox — is intrinsic to executing opaque, sender-provided payment wasm; it cannot be "fixed" without the penalty/metering machinery that created the 2.5 CSPR anchor in the first place. Given zero meaningful usage, the design friction and blocked 2.0 features (contract self-pay, prepay) are not justified. Removal is the design that best serves the 2.0 native-payment direction.
- **Alternative: keep Custom Payment, only lower the minimums.** Rejected: as long as Custom Payment exists, the failed-payment penalty must exist, and the penalty is what forces the effective minimum balance. The two changes are coupled; you cannot cleanly lower the minimum without removing the mechanism that anchors it.
- **Alternative: replace 2.5 CSPR with a different fixed constant.** Rejected: this would simply reintroduce a new magic number destined to drift from market reality, repeating the original mistake. A chainspec-controlled (and potentially governance-tunable) value is preferred.
- **Impact of not doing this:** the complexity drag on `contract_runtime` remains, advanced payment features stay blocked, and the minimum transaction cost stays artificially anchored to a pre-mainnet projection.

## Prior art

[prior-art]: #prior-art

The move away from user-supplied payment logic toward native, protocol-level fee handling mirrors the trajectory of other smart-contract platforms. Ethereum, for example, never exposed arbitrary "payment wasm" and instead handles gas payment natively at the protocol level (with EIP-1559 later making fee handling more dynamic and protocol-managed). Chains that expose minimum-balance / rent-style constants (e.g. Solana's rent-exempt minimum) have generally found value in making such thresholds protocol-configurable rather than hard-coded. The lesson consistently drawn is that fee and minimum-balance parameters should be adjustable via configuration/governance rather than frozen as source-level constants — which is the direction this CEP takes.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- What are the concrete default chainspec value(s) for the minimum execution balance and the minimum transfer amount at activation, and should they be a single shared value or two distinct settings?
- Should these minimums be a static chainspec value or wired to a dynamically calculated / on-chain-voted mechanism? (This CEP proposes chainspec as the baseline; a dynamic mechanism is left as a future possibility.)
- Are there any tooling, SDK, or client surfaces that still construct or expect Custom Payment wasm and would need deprecation notices ahead of rejection at the node?

## Future possibilities

[future-possibilities]: #future-possibilities

Removal of Custom Payment also removes the incompatibility blockage of more advanced payment options (contract self-pay, prepay, etc.).

It also removes the center of gravity holding down the 2.5 CSPR magical value. Beyond the chainspec-based minimum proposed here, this opens the door for lowering additional flat costs that were reasoned about relative to 2.5 CSPR.

It also falls neatly in line with changing the protocol to control minimum balance (for both execution permission, and new purse creation) via a dynamically calculated or voted upon value, rather than a static chainspec setting.
