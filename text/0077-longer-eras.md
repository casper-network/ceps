# Make ERA duration longer

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#77](https://github.com/casperlabs/ceps/pull/77)

The community came up with the actual problem :

- Staking rewards are too frequents and thus giving a hard time for users to track their rewards.

In some country with specific crypto regulation they ask to send the full history of transactions and with the actual ERA duration it creates a lot of staking rewards operations.

## Motivation

[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

Help users of the Casper Network to track more easily their rewards.
Also affects Validators.
Outcome expected : overall less rewards transactions and longer ERA duration.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

### User perspective

Currently, the ERA duration is 2 hours.
This creates 12 staking rewards operations per day and 4380 per year.
With the current number of delegators (71786 at the moment) it creates 314.422.680 rewards operations per year.

The current undelegation duration is 14h to 16h (7 era + current one).
The current delegation duration is 2h to 4h (1 era + current one).

We could increase the ERA duration to 6 hours with those others changes : 
- Set the undelegation duration to 2 era. This would effectively take between 12 and 18h to undelegate (2 era + current one).
- Keep the delegation period to 1 era. This would effectively take between 6 and 12h to delegate (1 era + current one).

In the best case scenario the undelegation will be reduced by 2h and in the worst case increase by 2h.

It won't probably cause any changes to the market giving how low the duration currently is and would be. 

In the end this will create 4 rewards operations per day and 1460 per year.

### Validators perspective

It will also modify how node operators handle different operations on their nodes.

It will increase the time needed to update the commission rate. (+)
- It will give more time to users to acknowledge the change. (Improvement proposed in CEP #61)

It will increase the time needed for small validators to move from non-participating to participating validators. (-)
- It will give more time to small validators to generate rewards.

It will increase the time needed for small validators to move from participating to non-participating validators. (+)
- It will give more chance for small validators to keep themselves in the top 100. 

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Set ERA duration to 6 hours.
Set undelegation duration to 2 ERA.
Set delegation duration to 1 ERA.

## Drawbacks

[drawbacks]: #drawbacks

Why should we *not* do this?

Other breaking changes not known at the moment ?

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?

- What is the impact of not doing this?
  - Nothing. We will still get a lot of operations per year.

## Prior art

[prior-art]: #prior-art

Other chains staking mechanism :

- Avax :
    - Delegators / Validators lock their token for a fixed time between 2 weeks and 1 year
    - Rewards are paid at the end of the staking term
- Elrond :
    - 1 reward per day. Claimable. Not compound.
- ETH 2.0 :
    - Reward period : 51d 12h. Rewards are locked until the beacon chain authorize withdrawals.
- Solana : 
  - Reward period : 2 day ~
  - Staking / Unstaking operations are fully commited at the end of the epoch (2 day ~)
- Cardano :
  - Rewards are compounded. Cardano epoch are 5 days each. Users earns rewards every 5 days after the first 20 days of approvals.
  - Users start getting rewards after 25 days and are paid 10 days later (2 epoch).
  - Unstaking is instant.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the CEP process before this gets merged?
- What related issues do you consider out of scope for this CEP that could be addressed in the future independently of the solution that comes out of this CEP?
  - The reward mechanism is being reworked for 2.0 I think as per the discussion in #65
## Future possibilities

[future-possibilities]: #future-possibilities

Make a full comparaison between all layer one blockchain and propose an updated model for Casper 2.0 ? 
