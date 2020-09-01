# Title

## Casper Node: Client Facing API

[summary]: #summary

CEP PR: [casperlabs/ceps#0009](https://github.com/casperlabs/ceps/pull/9)

This describes the client-facing API of the Casper Node.  It will be used by all clients interacting
with a Casper Node, including the Rust client, the Python client and Clarity.


## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

It will comprise two main parts: an HTTP server using [JSON-RPC](https://www.jsonrpc.org/specification)s
to satisfy client requests, and an HTTP server using
[Server-Sent Events](https://en.wikipedia.org/wiki/Server-sent_events) (SSEs) to satisfy streaming
events to subscribed clients.  The streamed events will be JSON-encoded data.

A shift in paradigm from the Scala node is that the node will not retain state relating to deploys
or blocks other than what is required to satisfy the correct operation of the node.  In other words,
the node will not retain state purely to be able to satisfy client requests to retrieve such
ancillary state.  Rather the node will present an event stream to which clients may subscribe,
allowing them to build up the information they require.

So, for example, rather than being able to return information like the cost of executing a deploy,
or an error message resulting from failed execution, this information will be announced by the
relevant component (the `BlockExecutor` in this case) and emitted via SSEs by the `SseServer`
component.

While this may complicate things somewhat for clients, perhaps especially for Clarity, it allows the
node software to be focused on its core responsibilities: accepting deploys, reaching consensus on
their order, executing them and persisting changes to global state and the blockchain.

Until we are able to provision some form of middleware which will be able to subscribe to nodes'
event streams, and to which clients can direct their queries for metadata around deploy execution,
the node will retain limited information in the deploy store as an interim measure.

This information will be held as metadata associated with a given deploy and will include the cost
of execution along with execution results.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The existing `ApiServer` will be renamed to `ClientRpcServer` and refactored to handle the JSON-RPC
portion of the API.

A new component named `SseServer` will handle client subscriptions and sending SSEs.  The
component's name specifically doesn't include the term "Event" to try and avoid confusion with the
existing multitude which relate specifically to the interaction between reactors and components.

The two servers will share the same listening address, with a path differentiating the two.

All messages from the node will include the API version.  This will allow the client to ensure it's
working against a compatible version of the node software.

### RPCs

The path for RPCs is `/rpc`

| Method               | Params                           | Success Response                                                 |
|--------------------- |--------------------------------- |----------------------------------------------------------------- |
| "account_put_deploy" | [JSON-encoded deploy]            | API version                                                      |
| "chain_get_block"    | [hex-encoded block hash] or none | API version + JSON-encoded block                                 |
| "state_get_item"     | { "block_hash": \<HEX STRING\> or null, "key": \<HEX STRING\>, "path": ["key1", "key2", etc], "type": \<STRING\> } | CLValue |
| "info_get_deploy"    | [hex-encoded deploy hash]        | API version + JSON-encoded deploy info                           |
| "info_get_peers"     | none                             | API version + list of node IDs (not validator IDs) and endpoints |
| "info_get_status"    | none                             | API version + list of peers + the latest block                   |

Note: For "chain_get_block", if no params are passed, the latest block is returned.

Note: For "state_get_item", if "block_hash" is null, the latest block will be used.  The "type" must
be one of "hash", "uref", or "address".  The returned value on success will be `CLValue`, serialized
using `ToBytes`, then hex-encoded.


### Event stream

The path for the event stream is `/events`.  It will send all events to all subscribers from the
moment they subscribe (i.e. no historical events will be sent).

the available event types are
* `block_finalized`
* `block_added`
* `deploy_finalized`
* `deploy_processed`
* `deploy_added`

The events corresponding to these subscriptions are as close as possible to those currently defined
in [info.proto](https://github.com/CasperLabs/CasperLabs/blob/cd2e80286d4172c9d2e73a2c1f1271b2961a7527/protobuf/io/casperlabs/casper/consensus/info.proto#L91-L161).

The first message returned will be the API version.  All subsequent ones will be JSON-encoded events.

### Use case

It's anticipated that the Rust client would be able to send a new deploy using the /deploy "put" RPC, while also having
subscribed to events filtered on that deploy's ID.

In this way, the client's `put-deploy` and `send-deploy` subcommands could accept arguments specifying that the client
wait until the deploy has transitioned to certain state.  Internally, the client will consume events (possibly reporting
each) until the requested deploy state has been reached, at which point the client can disconnect.

## Drawbacks

[drawbacks]: #drawbacks

* Clients will not be able to receive as rich information in response to requests for information as
they currently do, although this can be viewed as more of a trade-off than an actual drawback.


## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

* Plain JSON requests and responses via a REST API is a viable alternative to JSON-RPCs.  There
appears to be little difference in usability or implementation effort, although other projects in
the blockchain space use JSON-RPC, so perhaps there would be some slight benefit for us to follow
suit.

* Websockets could be used instead of SSEs.  These should work for us, but they provide full-duplex
connections, allowing back and forth messaging between the client and server.  Our use case is
simpler: the client specifies what it wishes to subscribe to in the GET request, and all further
traffic is from the node to the client.  As such, SSEs seems to be the simpler, better choice.

* Multiple event streams could be provided by grouping events into categories.  It doesn't seem like
this would be of any benefit, and if it results in a large increase of HTTP connections, could
instead be detrimental.

* Node could persist all ancillary information similarly to what is done in the Scala codebase.
This would increase the complexity of node and decrease the performance.  It would also consume
memory and disk resources.  It seems like a better solution to offload that to a separate process
designed specifically to handle that information.

* Events sent via the event stream could be filtered by the node rather than by the clients.  This
would be more convenient for clients, but places the corresponding burden on the node.  It may be
that future middleware can filter for clients, but for now it's better to keep the node's
responsibilities as minimal as possible.  This approach also allows clients to create filters as
coarse- or fine-grained as they need, rather than having the choice of filters imposed upon them by
the node.


## Prior art

[prior-art]: #prior-art

The existing Scala node is the best example of prior art.  While relevant in many aspects, the
proposed API here does not require persisting extraneous data purely to satisfy client requests for
that data.

There are also several other blockchain projects offering similar APIs, for example [Polkadot](https://polkadot.js.org/api/substrate/rpc.html).

## Unresolved questions

[unresolved-questions]: #unresolved-questions

None at this time.


## Future possibilities

[future-possibilities]: #future-possibilities

Kafka or similar has been proposed in the past, and with more emphasis on the node emitting an event
stream, perhaps this idea should be revisited in the future.
