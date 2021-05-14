# Synchronizer Redesign

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0048](https://github.com/casperlabs/ceps/pull/0048)

An improved design for the Highway synchronizer that better avoids redundant requests and balances load on the peers.

## Motivation

[motivation]: #motivation

The Highway protocol is based on nodes continuously merging their DAGs, but doesn't specify how this is done in detail. The synchronizer is the module that takes care of that: It translates between our internal protocol state and incoming and outgoing network messages, and implements the mechanism by which nodes exchange their DAGs' vertices.

We mainly want to optimize the synchronizer for the good case, to minimize latency: If I create a vertex I send it to everyone and assume that the recipients already have all units I cite; they usually won't need to download any dependencies and can add it directly to their protocol state, so the synchronizer's job is trivial.

However, we also want to handle cases as well as possible where messages got delayed, or, crucially, where a new node is joining the network and has to catch up and download the full protocol state of the current era. It also has to sometimes wait on the block validator component to check whether an incoming proposal contains a valid block.

The Highway implementation always broadcasts a new vertex if we just created it ourselves. It also always responds to requests for vertices from other nodes.

The synchronizer's job is to decide when to request dependencies, and from which peer. It has to find a compromise between:

* avoiding redundant downloads of the same vertex, and spamming a single peer with too many requests, and
* downloading dependencies as fast as possible to enable quick finalization of blocks.

The current implementation started off as a naive way of doing this, and has received some isolated improvements, but still makes redundant requests sometimes and is not as efficient as it could be.


## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The synchronizer keeps collections of pending vertices that cannot be added to the protocol state yet, because:

* their timestamp is from the future,
* they are missing a dependency, or
* they are waiting for validation of the block they contain.

Based on these collections, and the peers we received them from, we can deduce which missing dependencies we need to request, and whom we expect to have them: a correct node will only send us a vertex if it already has all dependencies.

To keep the synchronizer's memory usage small and its computations fast, we want to keep its state as small as possible, so we want to prioritize vertices in a way that minimizes the time we expect them to remain pending in the synchronizer: We always want to request those vertices first which are most likely to be added to the protocol state soon. That roughly means: first all evidence against faulty validators, since evidence never has a dependency and can always be added to the protocol state right away; then all endorsements and units, starting with those with an early timestamp, since they tend to have fewer dependencies.

Based on that prioritization we want to send requests so that:

* We request each vertex from only one peer at a time, but with an aggressive timeout.
* We don't request too many vertices from the same peer at once. The number will be configurable. A low default — even 1 or 2 — should be fine for a running node, but we should determine experimentally whether a higher value speeds up syncing for a joining node.

Finally, adding vertices to the protocol state as soon as possible is preferable over keeping them in the synchronizer queues: It allows full verification and earlier finality detection. So when syncing, we prefer to download the protocol state bottom-up: the earlier vertices before the later ones. That is why we will add a new way to specify a dependency, so that if we receive unit number 200 from a peer, and we only have unit 100 from that creator yet, we can request unit 101 instead of having to start from unit 199 and moving backwards.


## Reference-level explanation

We will move the synchronizer into the `highway_core` module, so it has access to the full content of each vertex, and all internal Highway data.


### New dependency type: "Unit with sequence number"

In addition to specifying units by hash, we add a message type

```rust
UnitBySeqNumber(u64, Hash)
```

which means we ask for a unit with the specified sequence number by the same sender as the unit with the given hash.


### Upon receiving a new vertex

If the vertex was requested from that peer, don't consider the request as pending anymore. (I.e. the sender is eligible to receive more requests.)

If it is a unit with a future timestamp, set a timer for that timestamp and keep it in the queue. When that timer fires, handle it again like a newly received vertex.

If it is missing a dependency or it is a proposal unit that still needs to be validated, keep it in the queue.

Otherwise add it to the Highway protocol state.

Finally, process the queue. (See below.)


### Upon completion of block validation

If the block could not be validated because all sending peers timed out, drop the vertex.

If the block is invalid, drop the vertex and everything that depends on it. Ban all the senders of those vertices.

If the block is valid, add the vertex to the protocol state.

Finally, process the queue.


### Processing the queue

Iterate over all pending units, sorted by their timestamp, up to the current timestamp, twice:

In the first pass:

* Check whether the unit depends on evidence we don't have, and request it if possible.
* If the unit does not have any missing dependencies at all, and is waiting for the block validator, make further block validator requests for all holders of that unit, for which no such request has been made.
* If it needs no further validation and has no missing dependencies, remove it from the queue and add it to the state.

In the second pass:

* If the unit's creator is not known to be faulty, check whether the unit's sequence number is greater than the minimal `s` for which we don't have any unit from the same creator with sequence number `s` in our protocol state: If it is, explicitly request `UnitBySeqNumber(s, _)` if possible.
* Check whether the unit depends on endorsements or other units we don't have and request them if possible. 

By "request if possible" we mean:

* Skip a dependency if a non-timed-out request is already in flight, or if we already have the vertex in the synchronizer queue.
* Skip if all peers that have the dependency (as far as we know) already have the maximum ongoing requests.
* Otherwise pick the peer with the lowest number of ongoing requests among those that have the dependency, and request the vertex from them.
* Start two timers: the first one aggressively (e.g. 1/10th of a minimum round length), and the second one generously (e.g. 10 minimum round lengths).

These iterations could take a lot of time if the queues are full, so we limit them to just the 1000 entries with the earliest timestamps: Since units cannot depend on other units with later timestamps, this can never cause us to be blocked.


### Upon timeout of a pending request

If it is the first (aggressive) timeout, from now on the request doesn't count as ongoing anymore for that _vertex_ (but still for the peer). Process the queue, which will usually cause the vertex to be requested from another peer.

If it is the second timeout, remove the peer from the set of holders of that vertex. If it is empty, drop the pending vertex.


### Limiting memory consumption

To prevent the synchronizer from consuming unlimited amounts of memory, we limit the list of pending vertices to a fixed number (e.g. one million). If that number is exceeded, we drop all pending vertices with the latest timestamps until we are below the limit again.

In practice, this should never be relevant unless a peer is trying to spam us: We prioritize units with lower timestamps, which are more likely to be added to the protocol state (and thus removed from the synchronizer queue) soon.


## Rationale, alternatives and prior art

[rationale-and-alternatives]: #rationale-and-alternatives


### Bottom-up synchronization

The synchronizer would have to keep much fewer pending vertices in memory if we managed to _only_ request vertices that we already know don't have missing dependencies. One way to do this would be to constantly keep track of all the peers' latest panoramas, and then to send them only vertices that we know they can add.

This would require a lot of additional messages for coordination, and could cause even more redundant requests: If everyone knows you are missing a vertex, everyone will send it to you. It's not clear how we could avoid that.


### Fully synchronize with random peer

Some DAG-based protocols (Hashgraph, PARSEC?) synchronize by periodically choosing a random peer and fully merging the two DAGs. This requires less synchronizer state. However, these protocols are asynchronous, and it's not clear this mechanism would satisfy Highway's synchrony requirements — or rather, it would only satisfy it with high probability if we have very long rounds.


### Keeping the existing implementation

We know from log messages that there are redundant requests, but we don't know how many. We could gather more data on this before committing to a new design.

But we also know that the existing synchronizer can potentially stall for a long time if a few peers fail to respond (e.g. because they go offline, or because our outgoing messages got dropped — which can happen if the connection breaks and gets reestablished).


### Do more research on DAG synchronization

We could try to find other synchronization methods in the scientific literature, and potentially improve the design before implementing.

A lot of our design is rather specific to Highway, though, and the design choices seem relatively straightforward.


## Future possibilities

[future-possibilities]: #future-possibilities


### Rolling average of peers' response time

Even better than a fixed timeout and request limit: Keep a rolling average of how long each peer takes to respond to dependency requests, and adapt timeouts and request counts accordingly.