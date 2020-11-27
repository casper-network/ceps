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

For points 1 and 2, we would define two new mechanisms:

- A `DisconnectAndBlacklist` request to the networking component (see [unresolved question 1](#unresolved-questions)).
- An `Accuse` message variant.

The `DisconnectAndBlacklist` request would cause the networking component to disconnect from the given node and add it to a blacklist, preventing it from reconnecting in the future.

The `Accuse` message variant would accuse the given node of malice and supply proof in the form of a signed invalid message. It would be gossipped like deploys or blocks, and nodes would disconnect and blacklist the indicated node when receiving such a message (unless the accusation turns out to be invalid, in which case they would disconnect from the accusing node and accuse it themselves).

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

We would also add the following to `protocol::Message`:

```rust
enum Message<I> {
    // ...other variants
    Accuse {
        node: I,
        invalid_message: Vec<u8>,
        signature: Vec<u8>,
    }
}
```

The `node` field would indicate the original sender of the `invalid_message`. The `invalid_message` should correctly deserialize into a signed message with invalid data, and the signature should be valid for the indicated sender. The `signature` should be a valid signature by the sender of the tuple `(node, invalid_message)`. If any of these conditions aren't satisfied, the accusation itself is invalid and the sender should be disconnected, blacklisted and (if the signature was valid) accused in turn.

## Drawbacks

[drawbacks]: #drawbacks

If we are too eager with punishments, we might blacklist nodes for issues that were beyond their control (corrupt data because of connection issues, for example) and break the network's liveness. However, the risk is small and can be mitigated by making sure that we only punish offences we are reasonably sure about.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The rationale has been explained [above](#motivation).

An alternative to the blacklist could be keeping some kind of local "reputation" score for nodes. This could be a more cautious approach and another way of decreasing the risk of excessive blacklisting. However, if we have a definitive proof of misbehavior, a blacklisting approach seems more preferable.

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

Currently, the messages exchanged between nodes aren't always signed. If we ever implement signatures in the network layer, we could get rid of the `signature` field in `Accusation` (as the message would be signed, anyway) and better identify the offenders when someone sends an invalid message.
