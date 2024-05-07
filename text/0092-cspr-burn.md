# CSPR Burning Mechanism

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0092](https://github.com/casperlabs/ceps/pull/0092)

Proposal to enable native token burning feature. 

## Motivation

[motivation]: #motivation

We want to make the existing internal CSPR token burn mechanism accessible to Smart Contract developers to let our community limit the Total Supply of CSPR tokens. This will allow us to introduce the deflationary mechanism to Casper Network. 

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Our proposal is;
- From a developer standpoint, allow developers to create a Smart Contract that can burn the CSPR tokens locked in it. 
- From a user standpoint, they should be able to decide to burn their CSPR token(s). 
- From the system perspective, when token(s) are burned, the Total Supply of CSPR token(s) should be reduced by an equal amount.


The system `mint` already has a mechanism to reduce the Total Supply of CSPR tokens, which has existed since pre-launch to support validator slashing. Even though slashing is currently turned off, the core capability exists, is integrated into the VM runtime, and is well tested. 

The purpose of this epic is to expose this functionality to smart contract developers by adding one new user facing FFI to the contract api and wiring it up to reduce a specified purse by the specified amount and use the mint’s existing method to reduce the total supply. Because this is leveraging existing capabilities, the level of implementation effort is low.


## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation



## Drawbacks

[drawbacks]: #drawbacks

It will introduce the deflationary mechanism and may impact the total supply of CSPR.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

A very important thing to list here is ideas that were discarded in the process, as these tend to crop up again after a while. Describing them here saves time as it allows people discussing those ideas again in the future to refer to this document.

**Rationale:**
- **Existing functionality**: The existing slashing mechanism in mint provides a tested and secure foundation for token burning. Reusing this functionality leverages existing code, reducing development time and potential risks.
- **Demand from developers**: Allowing smart contracts to burn tokens can unlock various use cases like token buybacks, governance mechanisms, limited-edition token offerings, and deflationary token models. Meeting this demand can attract developers and foster ecosystem growth.
- **Controlled implementation**: Exposing the functionality through a single FFI provides a controlled way to introduce token burning while maintaining system integrity.

**Alternatives:**

**1. Develop a new token burning mechanism:**
Pros: Allows for greater flexibility and customization compared to reusing existing functionality.
Cons: Increases development time, introduces additional code to maintain, and potentially creates new security risks.

**2. Use an external burning service:**
Pros: Offloads development and maintenance to a third party, potentially faster implementation.
Cons: Introduces reliance on an external service, which might have its own risks and limitations. Loss of control over the burning process and potential integration difficulties.

**3. Delay implementation:**
Pros: Allows for further analysis of potential impacts and gathering more user feedback.
Cons: May miss out on potential benefits and opportunities for developers and the ecosystem.

**Additional Considerations:**

- Governance mechanism: Define a clear process for approving and authorizing token burning within smart contracts to prevent misuse.
- Transparency and communication: Clearly communicate the rationale, limitations, and potential risks of exposing token burning to developers.
- Monitoring and evaluation: Continuously monitor the use of this functionality and its impact on the token supply and system stability.

Overall, reusing the existing slashing mechanism with a controlled FFI seems like a reasonable approach with low implementation effort and potential to benefit developers and the ecosystem. However, it's important to consider the alternatives and carefully weigh the pros and cons before making a final decision.

## Prior art

[prior-art]: #prior-art

**Prior Art for Exposing CSPR Token Burning to Smart Contracts**

**Good Prior Art:**

- Ethereum EIP-2535: This standard defines a function for burning ERC20 tokens within smart contracts. It provides a clear and well-established mechanism that can be used as a reference for CSPR's implementation.
- Cosmos SDK: The Cosmos SDK includes a module for token burning that integrates with its Tendermint consensus engine. This approach demonstrates how burning can be integrated with a secure consensus mechanism like CSPR's.
- Binance Chain BEP-20: Binance Chain's BEP-20 standard allows token burning within smart contracts. Studying their implementation can provide insights into potential challenges and best practices.

**Bad Prior Art:**

- Unsecured burning mechanisms: Some projects have implemented insecure burning mechanisms that were vulnerable to exploits or manipulation. Analyzing these examples can help identify potential pitfalls to avoid.
- Burning without proper governance: Projects that allowed unrestricted burning without proper governance mechanisms faced issues like inflation and token devaluation. Considering these examples can help establish a responsible approach for CSPR.
- Lack of transparency and communication: Some projects implemented token burning without clear communication to stakeholders, leading to confusion and mistrust. Prioritizing transparency can be learned from these negative examples.


## Unresolved questions

[unresolved-questions]: #unresolved-questions

**Governance and Approval:**
- Who can authorize token burning? Will it be restricted to specific smart contracts or open to any with proper authorization?
- What is the approval process for burning tokens? Is there a governance vote, committee approval, or automatic mechanism?
- Are there limits on the amount of tokens that can be burned at once or over a period?

**Technical Implementation:**
- How will the FFI function interact with the existing slashing mechanism? Are there any potential conflicts or unexpected behaviors?
- What are the potential gas costs associated with burning tokens? Will these costs be prohibitive for certain use cases?
- How will the burning process be reflected in the blockchain explorer and other monitoring tools? Can users easily track burned tokens?

**Economic and Security Implications:**
- What is the potential impact of token burning on the CSPR token supply and market price?
- Could excessive burning create deflationary pressure or hinder network adoption?
- Are there any security risks associated with exposing the burning functionality to smart contracts? Could malicious actors exploit it to manipulate the system?

**Developer Experience and Ecosystem Growth:**
- How easy will it be for developers to integrate token burning into their smart contracts?
- What resources and documentation will be available to support developers?
- How will this functionality attract new developers and projects to the CSPR ecosystem?

**Additional Considerations:**
- What are the long-term implications of introducing token burning?
- How will this feature be reviewed and updated in the future?
- How will the community be involved in discussing and making decisions about token burning?

## Future possibilities

[future-possibilities]: #future-possibilities

- Conditional burning: Allow burning based on specific conditions within the smart contract, like exceeding a funding goal or achieving a milestone.
- Batch burning: Enable burning multiple token types or amounts in a single transaction for efficiency.- 
