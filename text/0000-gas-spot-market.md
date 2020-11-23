# Spot market for gas

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0001](https://github.com/casperlabs/ceps/pull/0001)

The present CEP is intended to guide the development of the gas allocation mechanism that will be present at mainnet launch, 
later to be supplemented by a gas futures market and a fiat link. The CEP explains the concepts of willingness to pay (WTP), spot gas reservation,
gas allocation, gas requirements estimation, refunds and price determination. Further, it addresses the separation of block gas allocation for regular deploys and WASM-less transfers, 
balance checks and fallback purses. The present CEP is not intended to be a technical specification, but rather an explanation of the business logic
and the economic imperatives leading to the proposed deploy lifecycle.

## Motivation

[motivation]: #motivation

The Delta testnet that is active at the time this CEP is being drafted does not have any meaningful gas pricing, although it does charge
a nominal, hardcoded amount of motes per unit of gas. The reason we need to enable market pricing for gas is simply that gas represents
time spent on computing the changes to global state, thus making it an approximate measure of a finite resource. This resource is finite
because meaningful use of a blockchain requires imposition of what is effectively a soft time limit on block execution. Finite resources
must be priced both because it prevents spam attacks on the blockchain and because it is a means to discover the highest return uses
for the resource, which ultimately should be allocated to those willing to pay the most.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

In this section, we will outline the basic concepts and UX for blockchain users.

### Basic concepts

#### Willingness to pay (WTP)

Willingness to pay can be seen as analogous to "price" as informally used when discussing user bidding for gas. We will shortly
address why "price" should not be used for the deploy parameter we are about to define. Willingness to pay is almost self-explanatory, 
being the amount a user is willing to pay for a unit of gas. We are using the term pragmatically, not intending to mean that this is the
*maximum* a user is willing to pay. In fact, under our scheme, bidding is almost certainly going to be strategic, with users "shading" their
true willingness to pay, since the proposed allocation rule is analogous to a first-price auction.

#### Spot gas reservation

Spot gas reservation, conceptually, is the claim a user will lay on gas in a proposed block. Similarly to willingness to pay, it will be
a deploy parameter. The user effectively buys this amount, priced at the stated willingness to pay, when included in a block. To soften
the impact of poor gas requirements estimation, we will be implementing partial refunds for unused gas.

#### Gas allocation

Gas allocation with a greedy revenue-optimizing ordering of deploys is the core activity that this CEP proposes. It is envisioned that this
rule will be hardcoded into the block proposer component. The rule will be that highest WTP deploys with the highest spot gas reservations
be included first, subject to certain balance requirements. The charges arising from inclusion would be WTP multiplied by the spot gas reservation,
less any refunds arising from incomplete use of reserved gas.

#### What do we mean by "price?"

We have previously said that "price" should not be used when referring to the parameter we will be calling willingness to pay. The reason
for this is clarity of communication to external parties. Ultimately, "price" is a property of the market, not of individual discrete 
claims on a resource comprising the demand for the resource. "How much does it cost per unit of gas to include a reasonably sized deploy?" is a question
logically met with the answer "an epsilon above what the last deploy to make it into the last observed finalized block paid." However, because of uncertainty
at this margin, stating a higher willingness to pay leads to a higher probability of inclusion, a behavior users would expect from blockchains like Ethereum.
In fact, stating a willingness to pay above the observed "market price" directly amounts to paying extra for higher probability of inclusion, because of our allocation
rule.

### What does the user see?

The UI change for the user is the introduction (possibly by renaming) of a "willingness to pay" parameter and a "spot gas reservation" parameter. The UX
becomes somewhat more complex, since it is up to the user to determine the gas requirements. Because of our consensus-before-execution model, we are unable
to allocate gas on the fly. This means that to avoid underutilization of block gas, we must charge for the entire reserved amount (although with partial refunds, in recognition
of difficulties with estimating gas use of complex deploys). It is expected that the users, and particularly high WTP users, will make their best effort to estimate gas
requirements offline, or on our testnet, before submitting the deploys to the live network. We expect that, eventually, the community will develop tooling to make this estimation easier.
Some of the potential solutions may large dApps contributing developer time and nodes to maintaining a robust testnet that shadows complex contracts deployed on the live chain, adaptation of
our testing tools and application of empirical methods to observed gas use patterns.

### What does the protoblock proposer see?

The protoblock proposer, here a personification of the block proposer component, sees a deploy buffer, as usual. However, the block proposer will now be hardcoded to follow
the simple greedy rule we outlined above, alongside with balance checks to prevent attacks by "forgetting one's wallet at home" (we will refer to this, more formally, as the "nothing-in-purse problem").

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Chainspec parameters

#### Block gas limit

#### Share of gas dedicated to WASM-less transfers

### Deploy channels

### Lifecycle of a deploy

#### Ordering

#### Balance check

#### Allocation

#### Payment code execution

#### Session code execution

#### Refunds

## Drawbacks

[drawbacks]: #drawbacks

### Nothing-in-purse problem

### Payment code

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

### Account reputation system

### Pre-execution of payment code

## Prior art

[prior-art]: #prior-art

### Search ad auctions

### Ethereum pricing

## Future possibilities

[future-possibilities]: #future-possibilities

### Futures gas market

