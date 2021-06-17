# Incoming request quality-of-service

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

To ensure stable and fair service of requests even under load, we throttle incoming traffic from a peer to a fixed rate of requests. Peers are subjected to similar restrictions as for outbound bandwidth described in [CEP42](0042-network-qos.md), although instead of bytes we use a request count.

## Motivation

[motivation]: #motivation

By restricting the number of requests, we ensure a high quality of service (i.e. low block times), even when a lot of new users are joining the system. By reserving resources for active validators, this approach scales to a large number of newly joining nodes in case of a spike of interest in the network.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Each incoming request is subtracted from a per-peer limit (see [CEP42](0042-network-qos.md) for details). Should a node exceed this limit, incoming messages are delayed and no further data is read from the peers connection, causing regular backpressure through the underlying TCP/IP connection. It is important to note that in this case the burden of buffering requests is on the requesting peer, which is ideal.

Initially, only `GetRequest` messages, which typically make up the bulk of the traffic when joining, are subject to this limitation, although this can easily expanded to other network message types.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation


## Drawbacks

[drawbacks]: #drawbacks

A requirement to avoid a reduction in absolute maximum throughput is an upcoming reduction in memory usage that reduces the gap in resource consumption between large WASM-containing deploys and smaller variants. By reducing the impact of the former, a higher absolute request limit can be chosen, thus not jeopardizing the maximum throughput of transactions of the network.

Like [CEP42](0042-network-qos.md), this approach trades peak throughput for stability, with the current limiter, there is no mechanism to opportunistically allocate unused resources to improve the speed of new nodes joining.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The big alternatives are entirely different network designs, which are not discussed here, as they are out of scope. This change pushes the node to a more explicit validator-network-overlaid-over-general-network model.

## Prior art

[prior-art]: #prior-art

[CEP42](0042-network-qos.md) is the most prominent prior art in our own system.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

Some tuning of the numbers is inevitable and subject to occasional updates, as conditions change or improve due to changes in network's protocol. These questions will remain unanswered until determined and periodically refreshed empirically.

## Future possibilities

[future-possibilities]: #future-possibilities

* See [CEP42](0042-network-qos.md) for details.
