# Sequence number panoramas

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0056](https://github.com/casperlabs/ceps/pull/0056)

We propose citing units by sequence number instead of by hash.
This will reduce the protocol overhead and bandwidth requirement.


## Motivation

[motivation]: #motivation

32-byte hashes of earlier units make up the largest part of Highway messages.
Replacing them by 8-byte sequence numbers will therefore considerably reduce message sizes,
and improve performance.


## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Each unit — the main type of message in the Highway protocol — contains a list of hashes of earlier units.
That way the sender attests to which units they had received before they created the new one.
This is what defines the directed acyclic graph (DAG) structure of Highway's protocol state:
The graph has an arrow `u → v` if `u` _directly cites_ `v`, i.e. the message `u` mentions `v` by hash.

What matters to the protocol's logic is not the graph itself but the partial order it defines:
If `u → v → w` that means the creator of `u` must also have known about `w`, even if `u` doesn't directly cite `w`.
We write `u > v` if `v` is reachable from `u` by following any number of arrows.

In particular, the actual message `u` that gets sent over the wire does not need to _directly_ cite every `v < u`.
In the current implementation, `u` contains a _panorama_, which is a list containing for each (honest) validator the hash of the latest `v < u` created by that validator.

We propose replacing the hash with the _sequence number_ instead, i.e. the number of earlier units by the same validator.
If Alice has created units `a0`, `a1` and `a2` so far, and Bob wants to cite `a2`, instead of its hash, he will now write `2` in the panorama of his unit.

In the presence of equivocations the sequence number does not always uniquely specify a unit.
So whenever we cannot reconstruct the hash-based panorama, we fall back to requesting it from the sender.

[Panorama]: https://github.com/casper-network/casper-node/blob/b56c29cc97e86ead8d36dfdb2d41a8ecb1dffce9/node/src/components/consensus/highway_core/state/panorama.rs#L25


## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Units in memory still contain the full hash-based panorama to allow fast lookups.

But before sending a unit `u` over the wire we replace each hash referring to a `v < u` with `v`'s sequence number.
In addition, we add the hash of `u`'s full hash-based panorama so the recipient can verify after recomputing it.
And we add `u`'s own predecessor by hash, i.e. the previous unit by the same creator.

The unit wire format now looks like this (the first three fields are new, and the `panorama` field is removed):

```rust
struct WireUnit {
    seq_num_panorama: Vec<ObservationSeqNum>,
    panorama_hash: Hash,
    previous: Option<Hash>,
    creator: ValidatorIndex,
    instance_id: InstanceId,
    value: Option<BlockPayload>,
    seq_number: u64,
    timestamp: Timestamp,
    round_exp: u8,
    endorsed: BTreeSet<Hash>,
}

enum ObservationSeqNum {
    Correct(u64),
    Faulty,
    None,
}
```

`None` means that we haven't seen any unit from that validator yet, `Faulty` means they have equivocated.
`Correct(n)` is the direct citation of the validator's unit with sequence number `n`.

Upon receiving a unit, we request all missing dependencies from the sender.
This is now a bit more complicated than before, since we cannot request all units by hash anymore:

* As long as we don't have units with the required sequence numbers we request the missing ones, by sequence number.
* For validators with the entry `Faulty` we request evidence unless we already have it.
* If/when we have all units with the required sequence numbers, we use their hashes to reconstruct the hash-based panorama.
* If the reconstructed panorama's hash does not match the unit's `panorama_hash`, or if any validator is cited as `Correct` that we know to be faulty, we request the full hash-based panorama from the sender.
* Once we have the hash-based panorama, we request any missing units by hash, as well as endorsements (see section 3.6.1 of the [_Highway_ paper][Highway]) mentioned in the unit.
* Finally we add the unit itself to the protocol state.

The synchronizer implementation needs to be adapted in some places to support the new types of dependencies (unit by sequence number, hash-based panorama).

[Highway]: https://github.com/CasperLabs/highway/releases/tag/v2.0.2


## Drawbacks

[drawbacks]: #drawbacks

It will sometimes not be possible anymore to fully avoid redundant requests:
If e.g. we ask one peer for "unit with hash `0x123`" and another for "Alice's 5th unit", we don't know at that point if those refer to the same unit, and we will receive it twice.
This is in part addressed by [CEP 48][CEP48], but is also expected to be outweighed by the smaller unit size.

Another drawback is the increased complexity of the Highway implementation.

We plan to carefully test, review and measure the impact of this feature, and only release it if it is a significant improvement.

[CEP48]: https://github.com/casperlabs/ceps/pull/0048


## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

A potential alterative is to simply remove all redundant hashes from the panorama:
If `u` cites `v` and `v` cites `w`, don't include `w`'s hash in `u`, even if `w` is the latest unit by its creator.
This would require no changes to the synchronizer and be an overall simpler design.

However, witness units usually cite every validator's latest confirmation unit, and those confirmation units don't cite each other.
So for witness units this would be no improvement at all, unless we would also change the Highway schedule.

This CEP will reduce the size of almost every unit (except the earliest ones, that don't cite anything anyway).
Moreover, it provides the basis for even more effective panorama compression (see below).


## Prior art

[prior-art]: #prior-art

This CEP is based on an optimization described in section A.3 of [_Aleph: Efficient Atomic Broadcast in Asynchronous Networks with Byzantine Nodes_][Aleph].
Instead of a full hash, only a single bit is used per citation: whether that validator created a unit in the last round or not.

For Highway this approach has to be modified: Unlike in Aleph, you can cite units not only from the previous round but also from earlier rounds.

We want to thank the Aleph team for suggesting this and discussing its application to Highway with us!

[Aleph]: https://arxiv.org/pdf/1908.05156.pdf


## Future possibilities

[future-possibilities]: #future-possibilities

This feature leaves the door open for future improvements which can now be implemented by making only local changes to the code:

Instead of sequence numbers we can use _differences_ of sequence numbers:
If Bob cited Alice's message `a5` in his message `b10`, and now wants to cite `a7` in `b11`, he uses `7 - 5 = 2`.
Those differences will tend to be small; in fact, they will usually be 1, if validators are running at the same speed.

A list of small numbers with lots of ones is very compressible:
We can apply e.g. [Huffman coding][Huffman] or [RLE][RLE] to further reduce its size.

[Huffman]: https://en.wikipedia.org/wiki/Huffman_coding
[RLE]: https://en.wikipedia.org/wiki/Run-length_encoding
