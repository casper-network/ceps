# Equivocator Policy

CEP PR: [casperlabs/ceps#0038](https://github.com/casperlabs/ceps/pull/0038)

## Summary

[summary]: #summary

We propose a policy for dealing with faulty bonded validators. i.e. validators who have signed two mutually exclusive chain states. At main-net launch, such behavior will be treated the same as liveness failure, and faulty nodes will be temporarily suspended from consensus. Upon execution of a particular on-chain transaction by the node operator, the validator may once again be a consensus participant. 

## Motivation

[motivation]: #motivation

At the launch of CSPR, there could be operational issues due to software malfunction. For example, an operator may attempt to run multiple nodes using the same private key. Such behavior is strictly prohibited by the Highway protocol. In such cases non-faulty nodes will suspend processing signatures belonging to a faulty node.

But beyond the protocol's suspension, we need a general policy. We must address the following questions:

- How can node operators resume validating?
- What are the downsides for node operators when the network detects their node is faulty?

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The blockchain suspends faulty nodes until their operators reactivate them. The policy is exactly the same as nodes that disconnect from the network for an extended time (see [CEP 27][1]). A node operator can resume validating by sending a special transaction.

While suspended, a bonded validator node will not receive transaction fees or seignorage.

[1]: https://github.com/CasperLabs/ceps/blob/master/text/0027-disabling-bids.md

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

A special smart contract holds the bonded tokens used for validating. This is the auction contract. It controls bonded validator tokens and tokens delegated to them. We call a bonded validator's account in this smart contract a bid.

To reactivate a bid for a faulty node, a validator must do the following.

## Drawbacks

[drawbacks]: #drawbacks

The principal concern is an equivocating node or nodes might double spend. This is where two clients see inconsistent states. In this scenario, the attacker uses the same token for two off-chain transactions.

This attack is not practical. Official Casper labs nodes will shut down if they detect >33% of the bonded stake has equivocated. The network will then have to undergo an emergency restart as per [CEP 35][2].

Special care is necessary for light clients. A light client uses finality signatures rather than the Highway protocol. [CEP 26][3] describes finality signatures. Light clients should rely on the operating assumption that ≥67% of the stake in the network is not faulty. As a result, they will need ≥67% of the stake in finality signatures for maximal security against this attack.

[2]: https://github.com/CasperLabs/ceps/blob/master/text/0035-emergency-restarts.md
[3]: https://github.com/CasperLabs/ceps/blob/master/text/0026-finality-signature-gossiping.md

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The design proposed assumes that equivocation is most likely due to software malfunction. It leverages the existing design in [CEP 27][4] for handling network outages. It provides a path for node operators to recover.

In other designs, this path is not recoverable. Other proof of stake chains destroy validator funds in the scenario described here. This creates a lot of risk for validators.

[4]: https://github.com/CasperLabs/ceps/blob/master/text/0027-disabling-bids.md

## Future possibilities

[future-possibilities]: #future-possibilities

After a year, a governance system for the Casper blockchain will be in place. We expect to revisit the policy presented here when that time comes.