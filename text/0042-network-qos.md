# Network Quality-of-Service

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0042](https://github.com/casperlabs/ceps/pull/0000)

Introduce quality-of-service mechanisms for outgoing traffic to guarantee bandwidth for consensus.

## Motivation

[motivation]: #motivation

With a network growing in size, we want to guarantee that non-essential traffic (joining of non-validator nodes, gossiping for the benefit of non-validators) does not interrupt or degrade the actual consensus process.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

We are splitting the available upstream bandwidth (50 mbit, see below for estimates on these numbers) into two substreams, one for priority (validator) traffic and one for the reminder.

Every peer receives a score based on whether they are a validator (high), about to become a validator (medium) or neither (low). Peers are given guaranteed bandwidth from each according to their rank, peers scoring too low maybe also be disconnected.

Low ranking peers are also subject to rotation. New connections are still accepted and connecting higher ranks may evict lower ranks.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

For identification, a peer includes its validator public key and signs its peer ID, nonced by TLS server/client random. This information is optional, but allows the networking to know that the connection belongs to a particular validator.

Consensus (or another component) periodically announces the fingerprints of active and upcoming validators, allowing the networking to rank and allocate bandwidth accordingly.

Total bandwidth is configured by the node operator and divided into configurable slots. When the ranking changes, some clients may become evicted/disconnected. Lower ranks should periodically be cycled as well.

## Drawbacks

[drawbacks]: #drawbacks

Without dynamic bandwidth allocation this will reduce available bandwidth at light loads (see [Future Possibilities](#future-possibilities)); strong liveness guarantees require a large number of unused channels as well.

## Prior art

[prior-art]: #prior-art

## Unresolved questions

[unresolved-questions]: #unresolved-questions

* Actual size of protocol overhead.
* How do actual workloads factor in (e.g. gossip of deploys).

## Future possibilities

[future-possibilities]: #future-possibilities

Fixed bandwidth reservations constrain the nodes unnecessarily if connections are underutilized, e.g. when blocks are not full. A scheme like [Hierarchical Token Buckets](https://en.wikipedia.org/wiki/Token_bucket) could fairly simply be implemented to maximize bandwidth utilization without giving up the hard guarantees offered by static allocation.
