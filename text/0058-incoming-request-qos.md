# Incoming request quality-of-service

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0058](https://github.com/casperlabs/ceps/pull/0058)

To ensure stable and fair service of requests even under load, we throttle incoming traffic from a peer to a fixed rate of requests. Peers are subjected to similar restrictions as those for outbound bandwidth described in [CEP42](0042-network-qos.md), although instead of bytes we use a request count.

## Motivation

[motivation]: #motivation

By restricting the number of requests, we ensure a high quality of service (i.e. low block times), even when a lot of new users are joining the system. By reserving resources for active validators, this approach scales to a large number of newly joining nodes in case of a spike of interest in the network.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Each incoming request is subtracted from a periodically refreshed per-peer allowance (see [CEP42](0042-network-qos.md) for details). Should a node exceed this allowance, incoming messages are temporarily delayed and no further data is read from the peers connection, causing regular backpressure through the underlying TCP/IP connection. It is important to note that in this case the burden of buffering requests is on the requesting peer, which is ideal.

Initially, only `GetRequest` messages, which typically make up the bulk of the traffic when joining, are subject to this limitation, although this can easily expanded to other network message types.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The implementation itself is expected to reuse the existing facilities for this kind of rate limiting implemented with [CEP42](0042-network-qos.md). Since no deep introspection into the incoming messages is required, there is no large portion of business logic that needs to be moved into the networking component.

A single integer value determines the amount of allowance to subtract for a given message, which is realized as a method of the `Payload` trait. By setting this value to `0` or `1`, different message types can be in- or excluded.

## Drawbacks

[drawbacks]: #drawbacks

To avoid a reduction in absolute maximum throughput, an upcoming reduction in memory usage that reduces the gap between large WASM-containing deploys and smaller variants is required. Through this reduction, a higher absolute request limit can be chosen, thus not jeopardizing the maximum throughput of the network.

Like [CEP42](0042-network-qos.md), this approach trades peak throughput for stability. With the current limiter, there is no mechanism to opportunistically allocate unused resources to improve the speed of new nodes joining.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The alternatives involve entirely different network designs and are not discussed here. This change pushes the node to a more explicit validator overlay network model.

## Prior art

[prior-art]: #prior-art

[CEP42](0042-network-qos.md) is the most prominent prior art in our own system, otherwise rate limiting is common in other networks.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

Some tuning of the rates/numbers is inevitable and subject to occasional updates, as conditions change or improve due to changes in network's protocol. These questions will remain unanswered until determined and periodically refreshed empirically.

## Future possibilities

[future-possibilities]: #future-possibilities

See [CEP42](0042-network-qos.md) for details.
