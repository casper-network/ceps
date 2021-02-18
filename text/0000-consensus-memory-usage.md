# Reducing consensus memory usage


## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

Drop past eras' protocol states earlier, to use less memory.


## Motivation

[motivation]: #motivation

The network's validators are determined by an auction: If you want to become a validator, you send money — your bid — to the auction contract. You can add more money and increase your bid at any time. Withdrawing money reduces your bid immediately, but the payout happens with a delay of `U` eras, the _unbonding delay_. At the end of each era `E`, the highest bidders are selected as the validators for era `E + A + 1`, with a weight proportional to their bids. `A` is the _auction delay_.

So in era `E`, the validators' weights are proportional to staked tokens that will be paid out no earlier than at the end of era `E + U - A`. Thus `B = U - A` is the number of past eras whose validators are still bonded and can still be slashed, and whose signatures therefore can still be trusted.

Signatures older than that should never be trusted: The signers could have unbonded and sold their tokens, so they'd have nothing to lose. They could have sold their secret keys or decided to attack the network, knowing they cannot be slashed anymore, and the keys could have been used to sign a fork of the blockchain ("long range attack"). That is why joining nodes must always start from a trusted hash that is no older than `B` eras.

However, even within that period, signatures can only be trusted if we actually _do_ detect equivocations and slash validators. That's why the current implementation keeps the `B` most recent eras in memory and keeps syncing their protocol state: If someone attempted to mislead a joining node with a sufficiently recent trusted hash, they would have to equivocate in a recent era, and their equivocating units would be gossiped across the network and result in slashing.

But it gets worse: If we are in era `E`, not only does the above require us to keep in memory all eras back to `E - B`. It also means that any unit can depend on evidence from up to `B` eras ago. So era `E - B` could contain a unit referring to evidence of an equivocation in era `E - 2 * B`, and we still need to be able to validate that evidence. Fortunately evidence does not have any dependencies of its own, so the eras before `E - B` can be almost empty. ([HWY-257](https://casperlabs.atlassian.net/browse/HWY-257) will clear them.)

In a test with 75 validators, one block per minute, and an unbonding period of ~6 hours, this resulted in 310 MB of memory usage. HWY-257 should cut that in half, but it is still proportional to the unbonding period and to the _square_ of the number of validators. With 200 validators and an unbonding period of 1 day, that would already mean 4.4 GB of memory usage for consensus alone. And that is in the _good_ case, without any attackers trying to spam the protocol state.

In this CEP we propose a way to hold all still-bonded validators accountable without the need to remember all those old protocol states, greatly reducing the memory requirement.


## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

We drop an era's protocol state from memory as soon as all its blocks have a sufficient amount of finality signatures, and consider those signatures a replacement for the protocol state: just like the state, they are a proof of finality.

Conflicting signatures cause the signer to get slashed, so that — just like with equivocations in the Highway protocol state — the finality signatures are actually backed by the stake.

If an attacker creates an equivocation in an old era's Highway protocol, either it will have no effect, because everyone has already received the signed blocks and dropped the protocol state; or someone who still has the protocol state will detect the equivocation and still cause the attacker to get slashed.


## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation


### Slashing for conflicting finality signatures

Validators must create a finality signature for an executed block whenever they know with the configured `FTT` that that block is finalized, based on:
* a summit in the Highway protocol state with the required `FTT`, or
* valid finality signatures by `50% + FTT / 2` of the validators, by weight.

A finality signature is only valid if the signer also signed the parent block or wasn't a validator for the parent block (or an emergency restart; see below). Finality signatures get gossiped just like Highway units, and the sender of a signature is expected to also provide the signature of the parent block by the same signer on request.

With this rule, anyone who wants to validly sign two conflicting forks must create two finality signatures for two different blocks _at the same height_, and therefore create a compact proof of their misconduct. This proof, just like evidence from within the Highway protocol, is considered to satisfy the dependency of a block that contains the faulty validator in its `accusations`, causing the validator to get slashed.

If two conflicting blocks have `50% + FTT / 2` signatures, validators weighing at least `FTT` must have signed both blocks and produced evidence that can get them slashed.


### Replacing old protocol states with the signed blocks

The Highway protocol is used, as a separate instance in each era, to get validators to agree on that era's section of the blockchain. Whenever the protocol outputs a block as finalized, they create the finality signatures as a short proof of the block's finality (the in-protocol proof is quite complex). Therefore, once all blocks of an era have been finalized and signed by `50% + FTT / 2` validators (by weight), the Highway protocol state is not needed anymore, since we have a succinct proof of the consensus protocol's result.

Any node that has collected `50% + FTT / 2` finality signatures for all of the era's blocks can now delete the era's protocol state, and execute any remaining finalized blocks that it hasn't executed yet. To all requests for vertices in that era (other than evidence) it can respond with those signatures, so the recipient will also delete the protocol state.


### Finality is determined by finality signatures only

Once all nodes in the network have dropped the protocol state in an era, an attacker controlling a majority of the stake could now create an alternative protocol state: The equivocation could not be proven anymore, because everyone forgot about the previous instance.

That's why a node should only consider a block to be truly finalized once it has enough finality signatures. The in-protocol finalization based on Highway summits is only used as a trigger to create a finality signature.

That way, if the attacker wants to make us believe in their fork, they will have to create finality signatures, too, and for those they will still get slashed. If they don't produce those, the user won't see any finalized blocks and realize that they are not synced and connected to the live network.


### Emergency restarts

In the event of equivocation catastrophes, honest validators may end up signing conflicting forks. So the rule that the parent must also be signed does not apply here. After an emergency restart, you _are_ allowed to sign a different fork than before the restart, and cannot get slashed for it.


## Drawbacks

[drawbacks]: #drawbacks

TBD


## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Slashing for finality signatures needs to be implemented anyway at some point, and the rest of the solution is relatively simple and doesn't even use any disk space.


### Alternative: Swapping out whole eras to disk

Most old eras are inactive and don't change anymore, because nobody is creating units for them. We could swap them out to disk and just enqueue any incoming messages for them, and only swap them in (at most one at a time) when enough messages are waiting.

However, this CEP argues that it's not necessary to store them at all, which is simpler and less wasteful.


### Alternative: Backing protocol state by a database entirely

We could replace the `units` and `blocks` collections in `highway::State` with something backed by a database, that transparently caches the entries. In other words: We would _always_ store the protocol state on disk, and only keep the most recently used parts of it in memory as a cache.

That would also make the unit hash file obsolete, since even after a restart, we would know what units we produced.

The drawback is that it would make all the Highway algorithms — fork choice, finality detection, … — async (or even worse: blocking on I/O). It's not clear how that would fit in with the current structure of the code. And depending on the access patterns, which are hard to predict and can be influenced by an attacker, the algorithms' performance could change by orders of magnitude.


## Prior art

[prior-art]: #prior-art

TBD


## Unresolved questions

[unresolved-questions]: #unresolved-questions

None so far.


## Future possibilities

[future-possibilities]: #future-possibilities

This CEP doesn't rule out the second "alternative": In the future, we could still add a database backend for the protocol state in addition, reducing memory usage even further.
