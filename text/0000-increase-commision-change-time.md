# Increase time frame for commission

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

Currently changing the validator fees is very quick, this can catch all the delegators off-guard and provides a room for malicious actors to play the system to their advantage. 

## Motivation

[motivation]: #motivation

My intention behind this is 

  - Give time for delegators to assess the change when validator change commission fee %
  - Safe-guard casper community from bad actors

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

If the Validator chooses to change the commission % then he should allowed to do so only after a weeks notice.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Say i am a validator who needs to change my commission rate, then 

Step 1: I make my case by placing the new bid with my updated information. 
Step 2: cspr.live should display this new change in commission % for the community to see and make their judgement.
Step 3: This should be displayed for 7 days
Step 4: After this time is past, the new changes should reflect in the next ERA.

## Drawbacks

[drawbacks]: #drawbacks

I cant think of a reason why we shouldnt do this

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

There was a thought seeking communitys approval vote to accept reject the change in % and also for a min and max % but that would mean, too much of community engagement for too small a thing and moreover the onus is on the delegator to choose the right validator.

## Prior art

[prior-art]: #prior-art

The good: 

  - Create a safe community space
  - Provide the community an opportunity to take an informed decision

The bad:

  - Validators who have placed the bid by mistake will have to wait out the period for the correction to happen


## Unresolved questions

[unresolved-questions]: #unresolved-questions

  - Design to show the change is due in cspr.live with the exact time left to change
  
## Future possibilities

[future-possibilities]: #future-possibilities

Ideally i would want some sort of notification to be sent out to the delegators about this change.
I would like to develop something if the community desires. 
