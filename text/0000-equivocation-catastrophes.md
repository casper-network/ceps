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

The obvious first thing that should happen in the case of an equivocation catastrophe is that the software should alert the users about the situation. The node software is aware of the FTTs of the blocks, so when it detects equivocations, it can notice if the total weight of the equivocators exceeds the FTT of a block. When the FTT of any block is exceeded, the users should be made aware of it.

In order to make sure that as many validators as possible become aware of all the equivocations eventually, the operation of the nodes shouldn't stop. Finalizing blocks should be halted, as finalization ceases to mean anything when it can be reverted, but the nodes should continue gossiping votes.

The nodes could also output the hash of the last block whose FTT hasn't been exceeded and all the transactions included in the later blocks, and the list of equivocators. These will be useful for the manual steps.

The manual actions afterwards will consist of two main steps:

1. Reaching social consensus about the block to be used as the new starting point,
2. Restarting the network with the new starting point.

We will go into more detail about these steps below.

### Social consensus

For the initial version of this CEP, we assume that we have some kind of platform that enables open communication between validators and CasperLabs. The exact details of what that platform will be (e-mail? Discord? a forum?) can be left to be determined in the future.

The first essential step will be determining the block to be used as the starting point for the network after the restart. There are a few main options here:

1. It could be the last block whose FTT still exceeds the total weight of the equivocating validators - in which case it could be done automatically, as mentioned above.
2. Another option would be to use the tip of one of the forks of the chain, if more than one fork has been created before the detection of the equivocation catastrophe.
3. The last option would be to use a manually constructed block (more details below).

A desirable goal here would be to revert as few transactions from all the forks as possible, in order to limit the damage. Every transaction that was included in a block which appeared to be finalized for a time could have caused real world consequences (like, for example, a merchant sending ordered goods). Reverting any such transaction might potentially cause a loss to a user.

Option 1 essentially reverts all transactions that happened after the block whose FTT has been exceeded. Option 2 attempts to revert the transactions from all forks but one. However, it is possible that some of the transactions on different forks of the chain don't conflict with each other. In such a case, we could attempt to replay all of them on the new trusted branch. This is the idea behind option 3: to gather the non-conflicting transactions, create a block containing them and collectively declare it correct, outside of the consensus algorithm.

After the users collectively decide upon the last correct block, the transactions to be replayed and the list of equivocators to be slashed, the network can be restarted.

### Restarting the network

The process of restarting the network could be very similar to the process of starting it. In its essence, it is just like starting the network from scratch, but with a few differences:

- There is a different set of genesis validators, which is based on the bonded validators of the last correct block, with the equivocators removed.
- There is a different initial global state: it is not empty anymore, but it is the post-state of the last correct block.
- Other details like the protocol version can differ as well, but they are not of the main concern for the restart process itself.

The information necessary for restarting the network can (and probably should) be included in the chainspec. The "restart-chainspec" could be distributed among the validators the same way the initial chainspec for the network was.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Here we will just summarize the exact steps to be performed when an equivocation catastrophe happens.

1. An equivocation catastrophe is detected. Once a single validator detects it, we can expect all the validators to detect it eventually as well (even if just due to them receiving the evidence from the validator that detected it first).
2. Consensus gets suspended, though validators continue to gossip. The users (validator nodes' owners) receive notice about the catastrophe and start the process of social consensus.
3. The users agree about the last correct block, the transactions that need to be replayed on top of it and the set of equivocating validators (exchanging evidence if necessary).
4. A chainspec for the restart is manually constructed and agreed upon. This includes:
    - the initial set of validators for the restarted network which excludes the equivocators,
    - the initial global state hash for the restarted network
5. The network is being restarted using the chainspec constructed above.

## Drawbacks

[drawbacks]: #drawbacks

The single drawback is that this proposal requires manual intervention in the case of an equivocation catastrophe. An ideal situation would be if the network could handle these automatically, but this is most probably impossible.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

As mentioned above, automatic handling of equivocation catastrophes is probably impossible (as it would effectively mean constructing a consensus algorithm that exceeds proven theoretical limitations), so manual intervention is really the single option of dealing with such situations.

An alternative to social consensus would be to have a designated party deal with the situation. However, such a solution would go against the goal of creating a truly distributed system. In this alternative, the party responsible for handling equivocation catastrophes would effectively single-handedly control the network, which is something we want to avoid.

Not handling equivocation catastrophes at all would mean the end of the network once such a situation happens. Thus, even though we expect the probability of an equivocation catastrophe happening to be low (and we are making efforts towards making it as low as possible), the costs in the event of it happening are great enough to justify guarding against it.

## Prior art

[prior-art]: #prior-art

There have been cases of blockchain projects introducing manual interventions in order to revert catastrophical events, for example the Ethereum DAO hack.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- Which option for choosing the last correct block do we pick?
- Do we even require the last correct block, or do we just need to agree on an initial global state for the restarted network? Having the starting point for the new network in the form of a block would help in keeping the continuity of the blockchain and can be useful for history archiving purposes, but doesn't seem to be strictly required for keeping the functionality of the network.
- A related question which will be addressed in a separate CEP is about the process of joining the network by new validators. A new validator will have to be able to join the correct, agreed upon instance of the network, and might have to refer to social consensus in order to get the correct initial data (the correct chainspec).
- The details of the implementation will be defined once the questions here are fully resolved and will be laid out in a separate CEP.

## Future possibilities

[future-possibilities]: #future-possibilities

The process of social consensus can be used in the future for resolving other issues that cannot be dealt with automatically - however, what such issues can be, remains to be seen.
