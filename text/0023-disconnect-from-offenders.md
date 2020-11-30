# Handling nodes sending invalid messages

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0023](https://github.com/casperlabs/ceps/pull/23)

This CEP outlines the plan for handling nodes that send us invalid messages (binary blobs that don't deserialize to proper messages, or messages containing data that is invalid in other ways).

## Motivation

[motivation]: #motivation

Sending invalid messages, while relatively harmless (correct nodes will just reject them outright), is still a cost to the network - it uses up badwidth that could otherwise carry some valuable data. Simply ignoring such behavior would invite attackers to try to flood validators with garbage data in order to stall the network. Thus, we need to hold nodes sending invalid messages responsible for their actions. This CEP outlines a plan for doing exactly that.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

There is more than one way in which a message can be invalid:

- It can be just non-deserializable garbage.
- It can be deserializable, but the data contained in it can be invalid from the point of view of the handling component.

What it means for the data to be invalid depends on the component. For example, some kinds of messages contain binary blobs which can turn out to be garbage themselves. Or a message can contain references to hashes of some data and the hashes could point to data that doesn't exist on the network. There are many possibilities, which we won't even try to enumerate here - it suffices to say that a component can decide that a message is invalid and signal it somehow.

When we receive an invalid message, we know that the sender must be broken or malicious - correct nodes should validate messages before passing them on. Since keeping a communication channel to a malicious node is a risk, we will want to disconnect from such a node. We will also not want to let it reconnect to us, which implies holding a blacklist. Further course of action will depend on the type of invalidity:

1. If we cannot prove who sent the message to us, we stop here.
2. If we can prove that a given node sent the message (eg. because it contained a valid signature, but the signed data was invalid), we can tell other nodes by passing the message on to them as an accusation.
3. If we can prove that the sender runs a validator with a known key, we could slash them.

Determining what amounts to slash for what offenses should probably be done in another CEP. Currently, the only slashable offence is an equivocation and it is handled within the consensus component.

There is also a question of what is the adequate response to an offence for each of these 3 points. This CEP proposes that for offences not constituting a threat to the security of the network (eg., invalid messages that we can just detect and discard, as opposed to something like equivocations, which can cause problems to consensus), we shouldn't even bother telling other nodes - just disconnecting and blacklisting should be enough. More on that in the [Rationale and alternatives](#rationale-and-alternatives) section.

Thus, we would define a new mechanisms: a `DisconnectAndBlacklist` request to the networking component (see [unresolved question 1](#unresolved-questions)).

The `DisconnectAndBlacklist` request would cause the networking component to disconnect from the given node and add it to a blacklist, preventing it from reconnecting in the future. Specifically, it would blacklist the node's endpoint (IP and port). In order to keep the size of the blacklist manageable, we could consider various kinds of compression mechanisms and simplifications, such as blacklisting a whole IP address (all ports) if the same address shows up multiple times.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

We would add the following to `effect::requests::NetworkRequest`:

```rust
enum NetworkRequest<I, P> {
    // ...other variants
    DisconnectAndBlacklist { node: I },
}
```

The networking component would also keep a blacklist of nodes' endpoints (see [unresolved question 2](#unresolved-questions)). If a node attempts to connect from a blacklisted endpoint, the connection will be rejected (see [unresolved question 3](#unresolved-questions)).

## Drawbacks

[drawbacks]: #drawbacks

By choosing not to accuse nodes sending invalid messages to other nodes, we theoretically make it possible for a known malicious node to participate in the network for a while. However, the risk involved is rather small (see [Rationale and alternatives](#rationale-and-alternatives) below).

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Most of the rationale behind the proposal has been explained [above](#motivation).

One missing part is why not accuse offending nodes. The issue here is that accusation are a nonzero, even if small, cost to the network. A determined malicious party could trick the network into spamming itself with accusations by starting many nodes sending invalid messages. On the other hand, the risk associated with not immediately banning such a node is small. If an offender only sends valid messages to other nodes, there is no harm in still letting it communicate with the network, and if it tries sending an invalid one, it will just be blacklisted by the recipient independently. This means that we wouldn't be gaining much by accusing such nodes, but could potentially introduce a vector for a DoS-type attack.

As for alternatives, an alternative to the blacklist could be keeping some kind of local "reputation" score for nodes. However, if we are convinced of a node's misbehavior, a blacklisting approach seems more preferable.

## Prior art

[prior-art]: #prior-art

Banning for misbehavior is a common practice in the Internet communities, multiplayer games etc.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

1. Should the `DisconnectAndBlacklist` request be actually two requests: `Disconnect` and `Blacklist`? Would we ever want to disconnect without blacklisting?
2. Should the blacklist be a part of the networking component or a separate component? Since it probably wouldn't be used outside of networking, it would make sense for it to be a part of networking, as suggested.
3. Should the blacklist entries be endpoints (IP + port), IP addresses, node IDs, something else or a combination of the above?

## Future possibilities

[future-possibilities]: #future-possibilities

A possibility for future improvement involves making the blacklist more compact in order to reduce the risk of resource exhaustion attacks.
