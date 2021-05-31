# Protocol wire format negotiation

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0049](https://github.com/casperlabs/ceps/pull/0049)

Allow changing the underlying low-level wire format after handshake.

## Motivation

[motivation]: #motivation

The current low-level wire format of the node has inefficiencies that can be addressed in a backwards compatible manner, improving throughput.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

For backwards compatibility the casper-node has a low-level connection protocol that must not be altered:

* Clients connect to the server using TLS, establishing a bi-directional streaming transport. This is unaffected by this CEP and not discussed further.
* Client and server send a handshake simultaneously encoded using MessagePack (see below).

Nodes send the following fields in the handshake:

* network name
* public address
* protocol version (version 1.1.0 and up)
* consensus certificate (see [CEP42](0042-network-qos.md))

With the implementation of this CEP, an implicit wire protocol version is selected based on the protocol version field.

The wire format determines

* the kind of message being encoded,
* the framing, and
* the serialization format.

The underlying transport layer of TLS over TCP/IP is unaffected by this change. The existing wire format is specified to be version `1`, used for protocol version <= `1.3.0`. The handshake will always be transferred using wire format `1`.

After the handshake has been completed, nodes select the highest commonly understood wire format and change the format, if necessary.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The wire formats are defined as follows:

### Wire format 1

* Sends `Message`, which has two variants, `::Handshake` (0) and `::Payload(P)` (1).
* Length-prefixed frame, 4 bytes big endian length, chain-spec configurable maximum frame size.
* MessagePack encoded, as per [rmp-serde](https://docs.rs/rmp-serde/). Human readable, without field names, enum variants as integers.

An exception is introduced for the handshake: Its maximum frame size shall be 256 KB.

An example handshake can be visualized [here](https://sugendran.github.io/msgpack-visualizer/) (base64 input: `gQCSsnNlcmlhbGl6YXRpb24tdGVzdLExMi4zNC41Ni43ODoxMjM0Ng==`). Note the `"0":` part indicating the `Message::Handshake` variant.

### Wire format 2

* Encodes `P` directly without `Message` wrapper.
* Same framing as wire format 1.
* Uses [`bincode` 1.3.3](https://docs.rs/bincode/1.3.3/bincode) with [default options](https://docs.rs/bincode/1.3.3/bincode/config/struct.DefaultOptions.html).

Due to the more efficient binary encoding vs MessagePack human-readable, the resulting messages should be close to 50% smaller.

## Drawbacks

[drawbacks]: #drawbacks

Additional short-term implementation effort, larger testing surface.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Compression could be introduced reduce bandwidth, but comes with complication requiring batching/flushing of messages to achieve optimal compression & latency. Aside from large WASM payloads, most of our actual data is hashes, which should compress very poorly.

## Prior art

[prior-art]: #prior-art

The proposal is similar to [HTTP content negotiation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation).

## Future possibilities

[future-possibilities]: #future-possibilities

Should the need arise, more wire format versions can be introduced.

Changing `P` itself becomes possible, albeit at the cost of a major implementation effort, as `SmallNetwork` can likely no longer be generic.

This scheme may be rendered obsolete once fast sync has been implemented across the network, as there is no need to connect back to outdated nodes.
