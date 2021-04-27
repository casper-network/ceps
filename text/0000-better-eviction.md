# Improvements to faulty validator handling

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

Exclude validators that equivocated or were inactive in one era from all future eras,
even before the `auction_delay`.


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
they could choose among 2<sup>k</sup> leader sequences, even if those k nodes' stakes are small.

We propose a safe way to effectively ignore faulty validators anyway, even if the fault happened
after the auction, making it easier for the network to remain live without them.


## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The PRNG seed for an era is still determined by the block in which the auction took place.
And as a first step to compute the proposer sequence, that seed is still applied to the original
validator weights.

If after the auction a validator equivocates, it is still set to `Banned` status when the era is
initialized. In addition, we now also do this if the validator was inactive. I.e. whenever we know
that a validator is about to be evicted, we mark them as banned.

Now the actual proposer sequence is modified:
* If the proposer computation with the original set of weights returns a validator that was not
  banned, the validator retains their slot.
* If the computation returns a banned validator, the slot is reassigned by running the proposer
  computation with the modified set of validators, with all banned ones removed.

That way an attacker cannot gain more slots by making their nodes misbehave:
They only lose some of their own slots.


## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The first change to the protocol is that in addition to the equivocators, anyone who was reported
as inactive in a switch block since the auction is marked as banned as well, when a new era is
initialized. We will apply this only the the past `auction_delay` eras: If someone exhibited a fault
earlier than that, the auction contract will already have removed them.

The second change is a modification of the proposer slot assignment:
* We currently add (wrapping, as addition of two unsigned 64-bit integers) the era's seed to the
  slot's timestamp and seed a
  [ChaCha PRNG](https://docs.rs/rand_chacha/0.3.0/rand_chacha/struct.ChaCha8Rng.html)
  with the result, then apply it to select a validator, weighted by their stakes.
* Now, if the result is a banned validator, we use the same seed, but select a different validator,
  choosing only among the non-banned ones, weighted by only _their_ stakes.

Thirdly, we remove the banned validators' weights from the total weight for the purpose of fork
choice and finality detection: Apart from the proposer selection, the protocol behaves as if the
banned validators didn't exist.
This means the Highway instance has to keep track of both the original validator set and the one
with the banned validators removed, and use the first one only for the first step in the proposer
assignment.


## Drawbacks

[drawbacks]: #drawbacks

It makes things a bit more complicated.
However, we believe the approach is intuitive and straightforward.


## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

It would be simpler to just remove the banned validators altogether and just apply the PRNG to the
new weight map. However, that would reassign most slots, even those belonging to honest validators.
So an attacker with k nodes could choose among 2<sup>k</sup> different sequences by making any
subset of them misbehave. It is safer to only reassign their own slots.


## Prior art

[prior-art]: #prior-art

N/A


## Unresolved questions

[unresolved-questions]: #unresolved-questions

None


## Future possibilities

[future-possibilities]: #future-possibilities

It's tempting to not even wait until the next era, and reassign all future slots of an equivocator
immediately. However, who is seen as equivocating is subjective, so this will need a careful design
and analysis to make sure it does not affect liveness, and still leads to an objective definition
of a valid proposal unit.
