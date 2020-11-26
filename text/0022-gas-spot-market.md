# Spot market for gas

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0001](https://github.com/casperlabs/ceps/pull/0022)

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
Some of the potential solutions may be large dApps contributing developer time and nodes to maintain a robust testnet that shadows complex contracts deployed on the live chain, adaptation of
our testing tools and application of empirical methods to observed gas use patterns.

### What does the protoblock proposer see?

The protoblock proposer, here a personification of the block proposer component, sees a deploy buffer, as usual. However, the block proposer will now be hardcoded to follow
the simple greedy rule we outlined above, alongside with balance checks to prevent attacks by "forgetting one's wallet at home" (we will refer to this, more formally, as the "nothing-in-purse problem").

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Chainspec parameters

There are several natural parameters that govern the process we outlined. While it would reduce the scope of proposed work to leave these hardcoded, the optimal values cannot be determined in advance and will change as the platform grows, making it imperative that these parameters be included in the chainspec.

#### General gas limit

The limit on spot gas reservations by general deploys.

#### WASM-less transfer dedicated gas limit

The limit on spot gas reservations of WASM-less deploys, naturally stated as a multiple of desired number of such deploys multiplied by their (fixed) gas reservation. 

#### Refund proportion

Refund proportion for unused gas. Conceptually, this is equivalent to a discount on the unused reserved gas. The refund applies to total gas used, regardless of its use for payment or session code.

### Deploy channels

We have already mentioned the concept of a dedicated gas limit for WASM-less transfers in our description of the proposed chainspec parameters. To avoid crowding out WASM-less transfers, which are expected to be critical for platform health and public perception, there should be a separate "lane" for such transfers and this parameter specifies the width of this "lane." Selection among WASM-less transfers would work the same as it does for general deploys, but with sorting by WTP alone, since the gas reservation is fixed for these. Moreover, WASM-less transfers that are not selected for inclusion would be eligible for inclusion as general deploys, with stated WTP and fixed gas reservation.

### Lifecycle of a deploy

#### Construction

(user-side)

1. User creates a deploy in any supported manner, additionally specifying willingness to pay and gas reservation.
    
    a.  If the deploy is a WASM-less transfer, the gas reservation is fixed.

#### Submission

(user-side)

1. User submits the deploy to any node as usual.

#### Ordering

(validator-side)

1. Block proposer orders available deploys for inclusion in a protoblock.

    a. Sort by WTP (greatest to least).
    
    b. Sort by gas reservation (greatest to least).

#### Balance check & allocation

(validator-side, proposal)

For each channel, starting with WASM-less transfers, the following process is to be followed:

1. Traverse the ordered list of deploys, checking balances.
    
    a. Check that originating account purse holds an amount equal to willingness to pay times gas reservation, record the remainder for this originating account purse.
    
    b. If sufficient tokens, include the deploy in the protoblock.
    
    c. Upon encountering the same originating account purse subsequently, check willingness to pay times gas reservation against remainder, and update the remainder.
    
    d. If sufficient tokens in the remainder, include the deploy in the protoblock.
    
2. Continue traversing deploy-by-deploy until the gas limit is full.

3. Once gas limit cannot be filled by the next deploy with an eligible balance or remainder, skip deploys until a deploy small enough is encountered to be considered, otherwise continuing as above.

The remainder for each originating account is to carry over from WASM-less transfers to general deploys.

#### Payment code execution

(validator-side, execution)

1. Execute payment code as usual, but recording both originating account purse and purse provided by the payment code.

#### Session code execution

(validator-side, execution)

1. Execute session code as usual, but subject to the available tokens in both the originating account purse and the provided purse.

#### Charges & refunds

(validator-side, execution)

1. Charge the provided purse first.

    a. If no unused reserved gas and sufficient tokens, charge willingness to pay times gas reservation to the provided purse.
    b. If unused reserved gas and sufficient tokens, charge willingness to pay times used gas, plus willingness to pay times unused gas times refund parameter.
    
2. If the provided purse has insufficient tokens, charge overflow to the originating account purse according to the rule in 1.

## Drawbacks

[drawbacks]: #drawbacks

### Gas requirements estimation

We have already mentioned that, due to the consensus-before-execution model, we cannot charge only for gas used during execution. Charging only for gas used fails to provide any incentive for the users
to estimate gas requirements and open the proposed system to gaming by setting arbitrarily high gas reservations. This is contrary to the notion high-value transactions be prioritized and could possibly result in consistently underfilled blocks. However, this is a serious UX issue, since we have few tools at present for estimating gas requirements. It is expected that tooling will eventually be developed by third parties, using a testnet as a simulacrum of a live network. Such tooling may be developed from existing tests for simple contracts, but would require considerable work to enable testing of complex assemblages of contracts.

### Nothing-in-purse problem

The flexibility of our payment code presents a serious problem in the consensus-before-execution model, since the purse ultimately responsible for the charges cannot be checked until execution. Our proposal is seen to address this by requiring that the originating account act as a "soft" guarantor that deploy can pay for the reserved gas. We say that the originating account acts as a "soft" guarantor because uncertainty about the order of block inclusion in the linear chain means that, at execution time, the balance may no longer be sufficient in either the provided or the originating purses. However, we believe the balance check strategy we propose should prevent a sustained attempt to abuse the platform, since, eventually, the offending account will no longer be able to pass balance checks.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

### Second-price auction

One may observe that the proposed model of the spot gas market is essentially a first-price auction. While we do not currently know the strategic properties of our particular design, results concerning similar auctions suggest that it is not strategy-proof. In other words, users have to bid strategically, instead of merely reporting their *true* willingess to pay. We suspect that this would result in higher price volatility compared to a second-price auction (where charges would be calculated according to the *next user's* willingness to pay, rather than you own), due to user uncertainty about competition from other users. However, a second-price auction was ruled out, because the proposer-gets-all model of transaction fees enables validators to game such an auction by including own deploys in the protoblocks, potentially severely diminishing (or eliminating) consumer surplus.

### Account reputation system

We have considered assigning accounts reputation according to their observed ability to pay for deploys, but this was judged to be unworkable because of difficulty in designing reputation update rules that would enable new users to submit deploys, without leaving the platform open an attack by new unfunded accounts, thereby largely defeating the point of a reputation system.

### Pre-execution of payment code

We have considered executing payment code at proposal time, but judged that balance checks for originating account purses is a more elegant approach.

## Prior art

[prior-art]: #prior-art

### Search ad auctions

Our design is similar to the original ad position auctions used by search engines, before second-price position auctions were found to be preferable due to lower volatility. However, while similar in spirit, our auction ultimately allocates quantities, not positions, so we cannot provided any formal predictions about the properties or performance of our spot market at this time.

### Ethereum pricing

Our design is inspired by Ethereum, but adjusted to the realities of a consensus-before-execution model, primarily to avoid nothing-in-purse attacks.

## Future possibilities

[future-possibilities]: #future-possibilities

### Futures gas market

We expect that the spot gas market will be eventually supplemented by a futures gas market. Gas reserved on the futures market would then be used by deploys in a new deploy channel.
