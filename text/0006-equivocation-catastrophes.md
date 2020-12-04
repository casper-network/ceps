# Handling of Equivocation Catastrophes

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

This CEP describes a proposal for the process of handling equivocation catastrophes in Highway consensus (or, more generally: handling situations that break the assumptions behind the security of the consensus algorithm being used).

## Motivation

[motivation]: #motivation

Every consensus algorithm's safety guarantees rely on some assumptions being upheld. In the case of Highway, the main such assumption is about the validators not equivocating, that is: not producing conflicting votes. The algorithm can withstand some amount of equivocations, but if the total weight of equivocating validators exceeds a threshold called the Fault Tolerance Threshold (FTT), they could revert finalized blocks, which could lead for example to double spending. We call such a situation an "equivocation catastrophe" and it is something that cannot be handled automatically (as it would effectively contradict well-established results about the limitations of consensus algorithms in general). Thus, we require some kind of an external mechanism for dealing with such situations, should they ever happen.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The procedure for handling the equivocation catastrophes will have to consist of two main parts: what the software is expected to do in such a case, and what are the manual actions required to proceed.

The obvious first thing that should happen in the case of an equivocation catastrophe is that the software should alert the users about the situation. The node software is aware of the configured FTT, so when it detects equivocations, it can notice if the total weight of the equivocators exceeds the current FTT. When it is exceeded, the users should be made aware of it.

In order to make sure that as many validators as possible become aware of all the equivocations eventually, the operation of the nodes shouldn't stop. Finalizing blocks should be halted, as finalization ceases to mean anything when it can be reverted, but the nodes should continue gossiping votes.

The nodes could also output the hash of the last block finalized before FTT has been exceeded and all the transactions included in the later blocks, and the list of equivocators. These will be useful for the manual steps.

The manual actions afterwards will consist of two main steps:

1. Reaching social consensus about the new starting point,
2. Restarting the network with the new starting point.

We will go into more detail about these steps below.

### Social consensus

For the initial version of this CEP, we assume that we have some kind of platform that enables open communication between validators. The chosen platform should allow for storing, linking, referencing and displaying the whole public process and communication. We also should make sure that the content is in a format that can be displayed elsewhere.

The first essential step will be determining the starting point for the network after the restart. The suggested approach here would be to agree upon the new initial global state, new set of validators and restart the network as if it was starting from scratch.

A desirable goal for the initial global state would be to revert as few transactions from all the forks as possible, in order to limit the damage. Every transaction that was included in a block which appeared to be finalized for a time could have caused real world consequences (like, for example, a merchant sending ordered goods). Reverting any such transaction might potentially cause a loss to a user.

In order to minimize losses, the following approach is proposed: first, choose the last safe block as the starting point. A safe block would be one that still appears finalized with some small FTT, for example 1%, after a full synchronization of the protocol state.

After the block is chosen, take its global state and all transactions from the blocks that were considered finalized at some point, but are no longer a part of the main branch of the chain, and attempt to replay them on top of that global state. Since the order could matter, the community would have to agree on one (the default one could be based on deploy timestamps, but the community could decide on some other ordering if deemed necessary). The deploys' TTL could also be expanded to account for the time spent reaching social consensus and allow them to still be valid afterwards. Call global state resulting from replaying these deploys the "pre-initial state".

As the last step, agree on the initial set of validators for the restarted network. The equivocators should be slashed, but there may be reasons to also exclude other validators, or even to start with a set of validators that has nothing in common with the validators from the stopped network. Once agreement has been reached, apply necessary changes to the pre-initial state: modify the validator stakes and change the entries in the auction contract. After this is applied, the resulting state will be the state used for the restart.

After the users collectively decide upon the new initial state and the new initial set of validators, the network can be restarted.

Note that tools would have to be provided that would facilitate the manual construction of a specific state. This could be realized in the form of node software constructing the state based on the provided hash of a block, list of block or deploy hashes defining deploys to be executed on top of that block, and possibly some additional upgrade code.

### Restarting the network

The process of restarting the network would be very similar to the process of starting it. In its essence, it is just like starting the network from scratch, but with a few differences:

- There is a different set of genesis validators - the new validators chosen by the community.
- There is a different initial global state: it is not empty anymore, but it is the initial state prepared before.
- There is a different starting block height - we would not start from 0, but from the last safe block.
- Other details like the protocol version can differ as well, but they are not of the main concern for the restart process itself.

The agreed upon set of validators and the initial state would be included in the chainspec. The "restart-chainspec" (consisting of a new `chainspec.toml` along with any dependencies like `accounts.csv`) could be distributed among the validators the same way the initial chainspec for the network was. The files containing the new initial state could be distributed alongside the chainspec, or the nodes could download it from each other after they are started (which, of course, requires at least one node to possess the full copy of the state).

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Here we will just summarize the exact steps to be performed when an equivocation catastrophe happens.

1. An equivocation catastrophe is detected. Once a single validator detects it, we can expect all the validators to detect it eventually as well (even if just due to them receiving the evidence from the validator that detected it first).
2. Consensus gets suspended, though validators continue to gossip. The users (validator nodes' owners) receive notice about the catastrophe and start the process of social consensus.
3. The users agree about the new initial global state and the new initial set of validators. The new initial global state is manually constructed based on social consensus, using provided tools.
4. A chainspec for the restart is manually constructed and agreed upon. This includes:
    - the initial set of validators for the restarted network,
    - the initial global state hash for the restarted network (possibly given implicitly, in a form that will allow the node to construct it on the fly)
5. The network is being restarted using the chainspec constructed above.

Note that currently the chainspec doesn't contain the initial global state hash. Introducing such a field to the chainspec, allowing for the initial state to not be empty, and implementing global state synchronization between nodes would be required changes for the implementation of this CEP.

## Drawbacks

[drawbacks]: #drawbacks

However we try to account for all possible scenarios in this CEP, there can always be one exception that we have not accounted for. In that case, we would wait for it to play out, and eventually amend the CEP to account for the exception.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

As mentioned above, automatic handling of equivocation catastrophes is probably impossible (as it would effectively mean constructing a consensus algorithm that exceeds proven theoretical limitations), so manual intervention is really the single option of dealing with such situations.

An alternative to social consensus would be to have a designated party deal with the situation. However, such a solution would go against the goal of creating a truly distributed system. In this alternative, the party responsible for handling equivocation catastrophes would effectively single-handedly control the network, which is something we want to avoid.

Not handling equivocation catastrophes at all would mean the end of the network once such a situation happens. Thus, even though we expect the probability of an equivocation catastrophe happening to be low (and we are making efforts towards making it as low as possible), the costs in the event of it happening are great enough to justify guarding against it.

## Prior art

[prior-art]: #prior-art

There have been cases of blockchain projects introducing manual interventions in order to revert catastrophical events, for example the Ethereum DAO hack (see also [link](https://drive.google.com/file/d/1XuZ_3IcYbhg9hkPw2h6HH5rdkBsf2JS-/view)).

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- A related question addressed [in a separate CEP](https://github.com/CasperLabs/ceps/pull/5) is about the process of joining the network by new validators. A new validator will have to be able to join the correct, agreed upon instance of the network, and might have to refer to social consensus in order to get the correct initial data (the correct chainspec).
- The details of the implementation will be defined once the questions here are fully resolved and will be laid out in a separate CEP.

## Future possibilities

[future-possibilities]: #future-possibilities

The process of social consensus can be used in the future for resolving other issues that cannot be dealt with automatically - however, what such issues can be, remains to be seen.
