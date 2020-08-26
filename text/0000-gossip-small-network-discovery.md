# Gossip small network discovery

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

The `small_net` networking component has a built-in in mechanism for node discovery that should be replaced by using the tested, central gossping component.

## Motivation

[motivation]: #motivation

The current implementation for node discovery is fragile and not well tested, causing issue with nodes joining after having left the network. It is unclear whether or not the existing protocol as described in the module is free of race conditions.

This CEP suggest replacing a large portion of the `small_net` modules state and the entirety of its node discovery functionality with an instance of a `Gossiper`. The immediate advantage is the reuse of a well-tested and established component that is central to many other processes as well and reducing the amount of code actually inside the networking component.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Update the current implementation of `small_net` according to the following guide:

1. Each node starts up by connecting to one or more bootstrap addresses given at start-up.
2. Failed or closed outgoing connections are not retried.
3. When successly connected to a peer on an outgoing connection, add the outgoing connection to the internal connection map. This map is keyed by NodeID (derived from the public key presented), containing queues for outgoing messages.

   This is _all_ the connection-related state `small_net` retains.
4. All incoming connections are accepted, as long as they have a valid node id, but treated as stateless and distinct from outgoing connections (this is unchanged from the current implementation).
5. Every `n` seconds, each node gossips its own address.
6. Upon receiving a gossiped address, the node will attempt to connect to it once.

This proposal does away with *endpoints* entirely, as no node ever does outgoing connections "on demand".

### Example network lifetime



## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation




This is the technical portion of the CEP. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks

[drawbacks]: #drawbacks

Why should we *not* do this?

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

A very important thing to list here is ideas that were discarded in the process, as these tend to crop up again after a while. Describing them here saves time as it allows people discussing those ideas again in the future to refer to this document.

## Prior art

[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For development focused proposals: Does this feature exist in other applications and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your CEP with a fuller picture.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the CEP process before this gets merged?
- What related issues do you consider out of scope for this CEP that could be addressed in the future independently of the solution that comes out of this CEP?

## Future possibilities

[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the project as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the CEP you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future CEP; such notes should be in the section on motivation or rationale in this or subsequent CEPs. The section merely provides additional information.
