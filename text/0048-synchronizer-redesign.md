# Synchronizer Redesign and Sequence Number-based Dependencies

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

To keep the synchronizer's memory usage small and its computations fast, we want to keep its state as small as possible, so we want to prioritize vertices in a way that minimizes the time we expect them to remain pending in the synchronizer: We always want to request those vertices first which are most likely to be added to the protocol state soon. That roughly means: first all evidence against faulty validators, since evidence never has a dependency and can always be added to the protocol state right away; then all endorsements and units, starting with the earliest ones, since they tend to have fewer dependencies.

Based on that prioritization we want to send requests so that:

* We request each vertex from only one peer at a time, but with an aggressive timeout.
* We don't request too many vertices from the same peer at once. The number will be configurable. A low default — even 1 or 2 — should be fine for a running node, but we should determine experimentally whether a higher value speeds up syncing for a joining node.


## Reference-level explanation

We will move the synchronizer into the `highway_core` module, so it has access to the full content of each vertex, and all internal Highway data.


### New configuration options

We add new parameters to the Highway config:

* `request_short_timeout: TimeDiff` — If a peer does not respond this fast, we request the same dependency from another peer.
* `request_long_timeout: TimeDiff` — If a peer takes that long, we remove them from the set of holders.
* `max_requests_per_peer: u64` — How many parallel requests we make to the same peer.
* `max_pending_units: u64` — The maximum number of pending units that are kept in the synchronizer.
* `max_pending_endorsements: u64` — The maximum number of pending endorsements that are kept in the synchronizer.

The `pending_vertex_timeout` and `max_requests_for_vertex` options are deprecated.


### Events handled by the synchronizer

We now describe how the synchronizer handles events:


#### Upon receiving a new vertex

If the vertex was requested from that peer, don't consider the request as pending anymore. I.e. the sender is eligible to receive more requests — it doesn't count towards `max_requests_per_peer` anymore.

If it is missing a dependency or it is a proposal unit that still needs to be validated, keep it in the queue.

Otherwise add it to the Highway protocol state.

Finally, process the queue. (See below.)


#### Upon completion of block validation

If the block could not be validated because all sending peers timed out, drop the proposal unit.

If the block is invalid, drop the unit and everything that depends on it. Ban all the senders of those vertices.

If the block is valid, add the unit to the protocol state.

Finally, process the queue.


#### Upon timeout of a pending request

If it is the short timeout, from now on the request doesn't count as ongoing anymore for that _vertex_ (but still for the peer). Process the queue, which will usually cause the vertex to be requested from another peer.

If it is the long timeout, remove the peer from the set of holders of that vertex. If it is empty, drop the pending vertex.


### Processing the queue

First, for all pending endorsements, request the endorsed unit.

Next, iterate over all validators `v` with at least one pending unit, in random order. Choose a pending unit `u` by `v` with minimal sequence number `s`. Let `t` be the lowest sequence number such that we don't have a unit by `v` with it in the protocol state.

* If `u` requires an endorsement for a unit in our protocol state, request that endorsement.
* If the set of sequence numbers we have from `v` has a gap, request the lowest sequence number we don't have.
* If `s > t`, that means we just requested unit number `t`. In that case, continue with the next validator.
* If `s ≤ t` and the predecessor of `u` is not in our protocol state that means `v` must be faulty. Request the predecessor of `u` by hash and continue with the next validator.
* If we do have the predecessor, we can compute the sequence numbers of the panorama.
    * For every honest (as far as we know) validator `v'` where we don't have the required sequence number yet, request their earliest unit that we don't have.
    * If there are any validators for which we have multiple units with the required sequence number, request the full hash-based panorama of `u`.
    * If there are validators marked as faulty in `u` that we don't yet have evidence against, request the evidence.
* If we had to make any requests in the previous step, i.e. we don't have all matching units by sequence number from `u`'s panorama yet, continue with the next validator. Otherwise we are ready to guess the full hash-based panorama, and the checksum. If it doesn't match and we don't have `u`'s hash-based panorama yet, request it and continue with the next validator.
* If any citations are still missing from `u`'s hash-based panorama, request those, by hash, and continue with the next validator.
* If `u` is a proposal unit and needs validation, make block validator requests for all holders of `u`, for which no such request has been made yet.
* Otherwise add `u` to the protocol state.

When we write "request" above, we really mean:

* Skip if a non-timed-out request for the same dependency is already in flight, or if we already have the vertex in the synchronizer or the protocol state.
* Skip if all peers that have the dependency (as far as we know) are not free, i.e. already have `max_requests_per_peer` ongoing requests.
* Otherwise pick a free peer among those that have the dependency, and request the vertex from them. Start two timers: the short (`request_short_timeout`) and the long one (`request_long_timeout`).


### Limiting memory consumption and CPU usage

To prevent the synchronizer from consuming unlimited amounts of memory, we limit the list of pending units and endorsements to a fixed number (e.g. one million). Whenever that number is exceeded, we drop the unit with the highest timestamp among the units of the validator with the highest number of pending units.

Processing the queue could be quite slow if there are lots of pending units or endorsements, so the implementation either needs to index the set of pending units in a way that speeds it up, or limit the frequency with which processing happens.


### Catching up in a past era

If the other validators have already moved on to the next era we won't receive any new units from them, so nothing will trigger our requests for the remaining part of the protocol state. To address that, we keep the current mechanism of requesting our peers' latest units if:

* we haven't received any new vertex in a while, and
* our switch block is not finalized yet.


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
