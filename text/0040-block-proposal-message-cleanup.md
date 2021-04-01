# Block Proposal Highway Message Cleanup

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0040](https://github.com/casperlabs/ceps/pull/0040)

Remove redundant fields from the Highway block proposal messages.


## Motivation

[motivation]: #motivation

The block proposal message currently contains a `parent` field, which is redundant because the
parent is determined by the fork choice rule. It also contains identical `timestamp`s both as part
of the `value` field and as part of the `WireUnit`, as well as its own hash.


## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The Highway implementation is generic in the "consensus value" type, i.e. the payload part of the
block. In the concrete instance in production, this contains the deploys, transfers, accusations
(claims that a validator is faulty) and a random bit (for use in the seed of the PRNG that
determines the sequence of leaders).

It does not contain things like timestamp and height, since those are part of the structure of
Highway's protocol state itself.


## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Currently the consensus value is:

```rust
struct CandidateBlock {
    proto_block: ProtoBlock,
    accusations: Vec<PublicKey>,
    parent: Option<Digest>,
}
```

It contains a:

```rust
struct ProtoBlock {
    hash: ProtoBlockHash,
    wasm_deploys: Vec<DeployHash>,
    transfers: Vec<DeployHash>,
    timestamp: Timestamp,
    random_bit: bool,
}
```

We will replace these types with a single:

```rust
struct BlockPayload {
    deploy_hashes: Vec<DeployHash>,
    transfer_hashes: Vec<DeployHash>,
    accusations: Vec<PublicKey>,
    random_bit: bool,
}
```

The timestamp is already part of the message containing the payload for a proposed block, and the
parent and hash can be computed internally as needed; they shouldn't be part of the wire format.

The full Highway message containing a proposal is the following, serialized using bincode:

```rust
HighwayMessage::NewVertex(Vertex::Unit(
    SignedWireUnit {
        signature,
        hashed_wire_unit: HashedWireUnit {
            // hash: Not serialized; computed on deserialization.
            wire_unit: WireUnit {
                panorama, // The panorama, describing the new unit's justifications.
                creator, // The index of the block proposer in the list of validators.
                instance_id, // The unique identifier for the current era in this network.
                value: Some(BlockPayload {
                    deploy_hashes, // A list of deploys included in this block.
                    transfer_hashes, // A list of transfers included in this block.
                    accusations, // A list of public keys of faulty validators.
                    random_bit, // A random boolean.
                }),
                seq_number, // The number of previous units by the same validator.
                timestamp, // The time when this message was created.
                round_exp, // The round exponent. A round has 2.pow(round_exp) milliseconds.
                endorsed, // A list of hashes of endorsed units.
            },
        },
    }
))
```

…where a `Panorama` is a list of `Observation`s, one for each validator:

```rust
enum Observation {
    /// No unit by that validator was observed yet.
    None,
    /// The validator's latest unit.
    Correct(Digest),
    /// The validator has been observed equivocating.
    Faulty,
}
```

Note that this `HighwayMessage` is wrapped as the payload of a `ConsensusMessage::Protocol`.
There is one Highway instance per era, and this allows the message to get dispatched to the
appropriate one.
A `ConsensusMessage`, in turn, is wrapped in yet another message to distinguish it from message
types that don't belong to the consensus protocol, e.g. gossiped deploys.


## Drawbacks

[drawbacks]: #drawbacks

None


## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Keeping some duplicated entries in the messages would slightly simplify the implementation.
However, the protocol's wire format should not follow the implementation details of any particular
node software, and it is preferable to remove redundancy.


## Prior art

[prior-art]: #prior-art

None


## Unresolved questions

[unresolved-questions]: #unresolved-questions

None


## Future possibilities

[future-possibilities]: #future-possibilities

None
