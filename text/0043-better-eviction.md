# Improvements to faulty validator handling

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0043](https://github.com/casperlabs/ceps/pull/0043)

Exclude validators that equivocated or were inactive in one era from proposing blocks in all future
eras, even before the `auction_delay`.


## Motivation

[motivation]: #motivation

"Official" removal of a faulty (i.e. inactive or equivocating) validator from the validator set
happens with a delay:

In Highway, validators take turns at proposing blocks, and the sequence of proposers is determined
pseudorandomly. It is important that an attacker cannot manipulate the system in a way that allows
them to choose among a large number of proposer sequences and thus get more slots. That is why
the validator weights are determined in advance, with an `auction_delay` between the auction and
the beginning of the era: A small change to any weight would change the proposer sequence for
everyone, so it is important that before the auction, while bids can still change, the PRNG seed is
not yet known.

After the auction, during that delay, every block adds a single bit to the random seed that
will be used to initialize the PRNG. That way, an attacker, even if they had the last three blocks
before the new era's beginning, can choose only among eight different seeds, which are
all likely to be fair. If they could still e.g. reduce two bids with a million tokens each, they
could choose among a trillion different proposer sequences. Similarly, if an attacker has k
validator nodes and could remove themselves by letting any subset of nodes exhibit a fault,
they could choose among 2<sup>k</sup> proposer sequences, even if those k nodes' stakes are small.

We propose a safe way to effectively exclude faulty validators from proposing blocks anyway, even
if the fault happened after the auction, making it easier for the network to remain live without
them.


## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The PRNG seed for an era is still determined by the block in which the auction took place.
And as a first step to compute the proposer sequence, that seed is still applied to the original
validator weights.

If after the auction a validator equivocates, it is still set to `Banned` status when the era is
initialized. These banned validators are now excluded from the proposer sequence. In addition, we
also exclude validators that were inactive in a recent era. I.e. whenever we know that a validator
is about to be evicted, we exclude them.

Now the actual proposer sequence is modified:
* If the proposer computation with the original set of weights returns a validator that was not
  excluded, the validator retains their slot.
* If the computation returns an excluded validator, the slot is reassigned by running the proposer
  computation with the modified set of validators, with all excluded ones removed.

That way an attacker cannot gain more slots by making their nodes misbehave:
They only lose some of their own slots.


## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The first change to the protocol is that equivocators and anyone who was reported as inactive in a
switch block since the auction is marked as excluded when a new era is initialized. We will apply
this only to the past `auction_delay` eras: If someone exhibited a fault earlier than that, the
auction contract will already have removed them.

The second change is a modification of the proposer slot assignment:
1. We currently add (wrapping, as addition of two unsigned 64-bit integers) the era's seed to the
   slot's timestamp and seed a [ChaCha PRNG][1] with the result, then apply it to select a
   validator, weighted by their stakes.
2. Now, if the result is a excluded validator, we use the same seed, but select a different
   validator, choosing only among the non-excluded ones, weighted by only _their_ stakes.

**Example**: Alice, Bob and Carol have stakes 4, 3 and 5, respectively. However, Bob got banned.
We want to compute the proposer for round 42.
In step 1, we do the computation as if Bob were still taking part: We generate a random number
between 0 and 11, where 0, 1, 2, 3 would assign the slot to Alice, 4, 5, 6 to Bob and
7, 8, 9, 10, 11 to Carol. We initialize the PRNG with the era's random seed plus the round
number 42, roll the dice… and it's 6: The round would go to Bob.
In step 2, we realize that Bob is banned, so we now have to reassign the slot. This time, we
generate a random number between 0 and 8, where 0, 1, 2, 3 means Alice and 4, 5, 6, 7, 8 means
Carol. We apply the PRNG again: This time it's 5 and the slot goes to Carol.
She is this round's proposer.

[1]: https://docs.rs/rand_chacha/0.3.0/rand_chacha/struct.ChaCha8Rng.html


## Drawbacks

[drawbacks]: #drawbacks

It makes things a bit more complicated.
However, we believe the approach is intuitive and straightforward.


## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives


### Excluding faulty validators as if they hadn't won the auction

It would be simpler to just remove the excluded validators altogether and just apply the PRNG to the
new weight map. However, that would reassign most slots, even those belonging to honest validators.
So an attacker with k nodes could choose among 2<sup>k</sup> different sequences by making any
subset of them misbehave. It is safer to only reassign their own slots.

**Example**: Eve controls 20 cheap (low stake and/or much of it delegated) validator nodes that all
win the auction at the end of era 5. She also happens to be able to propose the last block of era 6,
with another validator node. Choosing the random bit in that block allows her to choose among two
seeds, and thus among two proposer sequences, for era 7.
However, if we went with the simple approach, she could now also decide to equivocate with any
subset of her 20 cheap nodes, and thus modify the validator set for era 7. This would allow her to
choose among 2 · 2<sup>20</sup> different seeds, i.e. select one of about two million different
proposer sequences. That gives her a greater chance of finding a sequence where she has a
disproportionate number of slots.

This is a rather expensive attack, of course, and the benefits don't seem to be that great, but it
still seems safer to defend against it by only reassigning the excluded nodes' slots.


## Future possibilities

[future-possibilities]: #future-possibilities


### Set excluded validators' weights to 0

We could remove the excluded validators' weights from the total weight for the purpose of fork
choice and finality detection. Apart from the proposer selection, the protocol behaves as if the
excluded validators didn't exist.
This means the Highway instance has to keep track of both the original validator set and the one
with the excluded validators removed, and use the first one only for the first step in the proposer
assignment.

The modified weights (with the excluded validators set to 0) also need to be applied outside
of Highway, wherever we count a block's finality signatures. Any clients that validate blocks should
use the modified weights as well.
That suggests it would be best to put the _modified_ weights into the block header, instead of the
original ones, or at least include all the information in every single switch block header that is
required to compute them. The original weights would still need to be communicated to the Highway
instance on initialization.

The advantage is that the faulty validators don't count towards the fault tolerance threshold
anymore, so the network would still be live if e.g. 20% started being inactive in era 4, and then a
further 20% started being inactive in era 5. On the other hand, this change would ignore the first
20%, even if they came back up in era 5.


### Exclude equivocators immediately, not just in the next era

It's tempting to not even wait until the next era, and reassign all future slots of an equivocator
immediately. However, who is seen as equivocating is subjective, so this will need a careful design
and analysis to make sure it does not affect liveness, and still leads to an objective definition
of a valid proposal unit.
