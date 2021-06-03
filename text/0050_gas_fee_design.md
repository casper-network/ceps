# Gas Fee Design

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0050](https://github.com/casperlabs/ceps/pull/0050)

This CEP proposes a framework for gas fee design. The framework aims to provide economic rationale and conceptual tractability.

## Motivation

[motivation]: #motivation

Gas fee was created by Ethereum, a Proof of Work (PoW) mechanism, to solve the issue of network congestion. As CasperLabs uses the Proof of Stake mechanism (PoS), the gas-fee design faces different economic incentives. The main difference is that PoW relies on gas fee to incentivize validation (mining), while PoS has the staking reward to compensate the validators. Therefore, the gas fee in PoS can target other incentive issues, like spamming attacks and storage maintenance. It follows that we need a custom gas-fee design for CasperLabs.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The proposed framework is an optimization problem formed by an objective function, a set of constraints and the associated optimal responses from the network users. The outcome is a policy function that maps the chosen features to gas fee, which is the purpose of the design.

#### Objective function
This is the institutional objective that maps the chosen network outcomes to a number valuation with an optimizer. For example, the objective can be choosing the gas fee to minimize the weighted sum of gas fee, network congestion and total storage. The objective function reflects the value and vision of CasperLabs. 

#### Constraints
The constraints reflect the technical and economic nature of CasperLabs ecosystem and regulate the range of of gas fee. For example, a set of constraints can be
* The congestion obeys a market demand function involving gas fee and market condition.
* The gas fee is high enough to compensate the computation on average so the validators don't shirk.
* The gas fee is high enough so the potential spammers don't attack the network.
* The benefit of clearing non-valuable storage overweights the benefit of creating a new account.

#### Optimal responses
These are the decisions from the users' perspective, which are part of the objective or constraints. Following the above example, the optimal responses are
* A spamming function that is solved from the spammer's problem.
* A deletion decision that is solved from the user's problem.
Note that we may use reduced-form function to simplify the problem if we believe the responses are not highly strategic, e.g. we use a market demand function instead of aggregating the demand from individual responses.

#### Policy function
This is the end result, a pricing function governs how much gas fee a transaction should pay. The function takes three kinds of features
* Transaction specific features, e.g. the size/difficulty
* Account specific features, e.g. the storage and age
* Market conditions, e.g. the congestion/capacity
All these features are ingredients for the objective and constraints. Since the policy function is the solution of the optimization problem, we have the rationale built in it through the choice of objective function.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The framework is a straight-forward optimization problem. However, depending on the function form, the solution might not be intuitive. Nonetheless, we can apply a piece-wise linear approximation so the calculation is user-friendly. Given the above incentives, the typical gas fee is
* Increasing in the computation difficulty
* Increasing in the account storage
* Decreasing in the account age
* Increasing in the congestion
The specific results depend on the functional form we use in the framework.

## Drawbacks

[drawbacks]: #drawbacks

The contingent pricing for gas fee is not a status quo application. An unfamiliar user experience can affect the network adoption. Therefore, we should keep things simple and intuitive. In the end, we don't just target the existing blockchain community but all people and organizations. 

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The framework is good for three reasons.
1. It is explicit about the institutional goal, reflected in the objective function.
2. It calculates the optimal gas fee given the assumptions.
3. It allows easy computation and directional updates in the future.

The alternative is to add adjustments to the status quo (Ethereum model), which is worse because of the incentive difference between PoW and PoS. Moreover, such alternative doesn't reflect our vision and value.

## Prior art

[prior-art]: #prior-art

The current Ethereum gas fee involves a first-price auction. The rationale is to promote the most valuable transaction under congestion. It's challenging for the average users to solve the auction problem and the timely auction is bad for cost stability. Therefore, Ehtereum proposes to transit to [EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md) that has three components, a base fee, a tip and a fee cap. The formula is (a bit) ad hoc. The implementation is scheduled for July, so the effect is unknown for now.

On the other hand, to keep storage low, Ethereum introduced gas refund for storage deletion. However, the refund mechanism actually increased the network storage, due to the speculation on gas fee. The proposal suggests that conditioning gas fee on the account storage is a simple and intuitive solution to keep storage low. The key idea is to keep the user decision static (current fee depends on current storage) instead of intertemporal (future fee affects today's storage).


## Unresolved questions

[unresolved-questions]: #unresolved-questions

The proposed framework is abstract from the specific functional forms. To implement it, we need to determine the components. Moreover, the function parameters may require empirical studies.

At the meantime, the gas-fee design interacts with the staking auction and gas future. An integrated overview is necessary. The recommendation is to use each instrument to solve a distinct part of the overall problem, e.g. staking auction targets the validator participation, gas fee targets the user behavior and gas future stabilizes the network expectation.

## Future possibilities

[future-possibilities]: #future-possibilities

As the network adoption is in its early stage, we should expect a changing environment. It follows that the optimization problem needs to be updated and resolved overtime. On the bright side, the framework itself is directional and reusable.
