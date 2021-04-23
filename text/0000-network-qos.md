# Title

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

Introduce quality-of-service mechanisms for outgoing traffic to guarantee bandwidth for consensus.

## Motivation

[motivation]: #motivation

With a network growing in size, we want to guarantee that non-essential traffic (joining of non-validator nodes, gossiping for the benefit of non-validators) does not interrupt or degrade the actual consensus process.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

We are splitting the available upstream bandwidth (50 mbit, see below for estimates on these numbers) into equal-sized substreams, at least 1.5X the number of validator slots.

Every peer receives a score based on whether they are a validator (high), about to become a validator (medium) or neither (low). Peers are given guaranteed bandwidth from each according to their rank, peers scoring too low are disconnected.

Low ranking peers are also subject to rotation. New connections are still accepted and connecting higher ranks may evict lower ranks.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

For identification, the a peer includes its validator public key and signs its peer ID, nonced by TLS server/client random. This information is optional, but allows the networking to know that the connection belongs to a particular validator.

Consensus (or another component) periodically announces the fingerprints of active and upcoming validators, allowing the networking to rank and allocate bandwidth accordingly.

Total bandwidth is configured by the node operator and divided into 0.33 mbit slots. When the ranking changes, some clients may become evicted/disconnected. Lower ranks should periodically be cycled as well.

## Drawbacks

[drawbacks]: #drawbacks

Without dynamic bandwidth allocation this will reduce available bandwidth at light loads (see [Future Possibilities](#future-possibilities)); strong liveness guarantees require a large number of unused channels as well.

## Prior art

[prior-art]: #prior-art

### Bandwidth estimates

* Full blocks at 100 validators result in having to send roughly 2800 hashes at 32 bytes to each, in 33% of the round time.
* A round is 64 seconds, leaving ~ 22 seconds for this process.
* We include 10% margin for overhead from network encoding and TCP/IP, an almost unrealistic best case.
* We assume a 50mbit connection (10 mbit is too small, but 100mbit across the world seems hard to guarantee).

This results in a bandwidth requirement of $\frac{2800\cdot32\cdot100\cdot1.1\cdot22}{1024^2\cdot8} \approx 3.4 [\textrm{mbit}]$ for just transferring hashes and consensus, 100 KB transferred per validator. We divide by 150 slots and can transfer roughly 2.5 megabytes of additional bulk data per peer to cover gossip, address and protocol overhead, as well as bulk data sent directly. We are relying on gossiping to spread the load here, or the network will slow down.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

* Actual size of protocol overhead.
* How do actual workloads factor in (e.g. gossip of deploys).

## Future possibilities

[future-possibilities]: #future-possibilities

Fixed bandwidth reservations constrain the nodes unnecessarily if connections are underutilized, e.g. when blocks are not full. A scheme like [Hierarchical Token Buckets](https://en.wikipedia.org/wiki/Token_bucket) could fairly simply be implemented to maximize bandwidth utilization without giving up the hard guarantees offered by static allocation.
