# Gossip small network discovery

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#10](https://github.com/casperlabs/ceps/pull/10)

The `small_net` networking component has a built-in in mechanism for node discovery that should be replaced by using the tested, central gossiping component for more robustness and simpler implementation.

## Motivation

[motivation]: #motivation

The current implementation for node discovery is fragile and not well tested, causing issues with nodes rejoining after having left the network. It is unclear whether or not the existing protocol as described in the module is fundamentally free of race conditions that would prevent a network from being eventually fully connected.

This CEP suggest replacing a large portion of the `small_net` components state and the entirety of its node discovery functionality with an instance of a `Gossiper`. The immediate advantage is the reuse of a well-tested and established component that is central to many other processes as well and reducing the amount of code actually inside the networking component.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

`small_net` is updated as follows:

1. Each node starts up by initiating a connection to one or more *bootstrap* addresses set through the configuration.
2. When successly connected to a peer through an outgoing connection, it adds the outgoing connection to the internal connection map. This map is keyed by NodeID (derived from the public key presented), containing queues for outgoing messages.

   This is _all_ the connection-related state `small_net` retains.
3. Failed or closed outgoing connections are not retried and their respective entries remove from the connection map.
4. All incoming connections are accepted, as long as they have a valid node id. They are treated as stateless and distinct from outgoing connections, triggering an announcement on a received message (this is unchanged from the current implementation).
5. Every `n` seconds, each node gossips its own address.
6. Upon receiving a gossiped address, the node will attempt to connect to it once.

This proposal does away with *endpoints* entirely, as no node ever does outgoing connections "on demand", that is after being requested to send a message to specific node. The term *root node* should also be deprecated -- the network was always intended to be decentralized and calling a particular node a root node is confusing.

(Historically, the root node was the first node whose address was known. To avoid potential bugs in practice all nodes connected to this node initially, which caused it to always have an up-to-date address list. With the update in this CEP, this should be no longer the case or necessary.)

### Example networking scenario

An example scenario is laid out here:

1. The first node, `A`, starts up, without any bootstrapping nodes. It just opens a listening port, waiting for connections.
2. `B` and `C` are started, with `A` as the bootstrapping node. They both connect to `A`, thus can now send messages to `A`, but will not receive any from it.
3. `A` gossips its address, causing no change, since `B` and `C` already know `A`s address from bootstrapping.
4. `B` and `C` gossip their addresses, causing `A` to connect to `B` and `C`, `B` to `C` and `C` to `B`.
5. The network is now fully connected.
6. `D` connects with `B` and `C` as bootstrapping nodes.
7. `B` drops from the network.
8. `A` gossips its address, causing `D` to connect to `A`.
9. `D` gossips its address, causing `A` and `C` to connect to `D`.
10. The network is again fully connected, with nodes that want to send messages to `B` eventually dropping them if they have not detected its failure yet.
11. `B` rejoins, connecting to `A`.
12. `B` gossips its address, but the gossip does not reach `C`. This causes `A` and `D` to connect to `B`.
13. The network will be fully connected once a repeat gossip from `B` reaches `C`.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The necessary changes to the `small_net` component include the removal of the `Endpoint`, to be replaced with just an address. Identity of nodes is established on connection securely via TLS --- remember that a nodes NodeID is just the hash of its public key and cannot be faked without knowledge of the secret key --- and the existing scheme does nothing to prevent against Sybil attacks or DDoS'ing by spamming new node announcements.

Connection upgrades are handled as regular events instead of separate tasks to serialize updates to the connection mapping. The connection mapping itself is a bi-directional mapping, as we are maintaining exactly one outgoing connection per NodeId and need to remove them by NodeId as well.

Any connection is tagged with an internal, monotonically increasing ID to disambiguate it, new connections with a lower ID than an established connection are ignored/closed immediately. Similarly, close/fail events only remove connections with their respective IDs. Even in the case of completely random reordering of all events, this ensures that at worst we track a dead connection to a node for a while (in the case where a connection is closed immediately and the close event overtook the open event).

Connections are "naturally" garbage connected by being selected for gossiping; upon trying to send a message to a closed connection, a "connection closed" event is generated if the outgoing message fails.

## Drawbacks

[drawbacks]: #drawbacks

The previous discovery implementation aimed for an instant, complete and reliable full connection between all nodes, while this replacements delays this by up to $2 (n + \textrm{gossipDelay})$.

Additionally it increases the burden on the network by gossiping addresses, instead of storing them on each node.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

This design was chosen over the existing implementation to reduce the testing burden, making it easy to prove its eventual correctness based on an already existing tested component, the gossiper.

## Prior art

[prior-art]: #prior-art

Very few prior art (except the existing implementation) was considered for this, as this is a short-term improvement for the networking layer.

## Future possibilities

[future-possibilities]: #future-possibilities

The future for the networking layer is likely a complete replacement with either an off-the-shelf solution or a custom implementation that does not need to maintain a fully connected network with two TLS connections per graph edge. This is out of scope of this CEP.
