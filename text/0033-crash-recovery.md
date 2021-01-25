# Node Crash Recovery

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0033](https://github.com/casperlabs/ceps/pull/33)

This is an outline of a plan to allow nodes to rejoin eras that are already in progress, without risking to equivocate.

## Motivation

[motivation]: #motivation

Validators sometimes stop participating in the network. They might do it deliberately, or it can happen because of network outages, or because the user just turned their computer off. If too many validators go offline at the same time, the network can stall. It should be possible for the network to recover when enough validators come back online - currently it is not, though.

What we currently do is we only allow the rejoining validators to start voting once a new era begins. This is designed to prevent accidental equivocations - if a validator crashes, they might not have saved any information about some messages they have sent, and they could try to create new units in the consensus DAG that would conflict with some others created before the crash. In such a case, other validators would consider it malicious.

However, if a network is stalled, the new era won't be able to begin, because there aren't enough validators to finish the current one. Thus, even if enough validators come back online, this currently doesn't help: they won't vote until a new era begins, and this will never happen.

This CEP aims to propose a solution that would allow the validators to start voting in eras that are already in progress, making it possible for the network to resume normal operation after temporary issues.

**Note 2020-01-11:** We currently have a quick workaround in place, which basically does what is described below, except it doesn't cater to endorsements, nor does it use storage - it bypasses the storage component by directly creating additional files. While working in practice for the moment, it should be replaced by a proper solution.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Since the main problem is the risk of creating an accidental equivocation, we need to make sure that a rejoining validator will be aware of all its previous messages in the relevant era.

For the units, it is enough to save the hash of the latest created unit. This would make it possible to download the unit from other validators along with its whole history and be able to verify the correctness of downloaded data.

Units aren't the only thing that might be used to create an equivocation in Highway, though. Endorsements are such a thing, too. It would therefore be necessary to save the hashes of all endorsed units, too. When rejoining the era, the validator would read the list of such hashes, download the relevant units and make sure that it never endorses conflicting units in the future.

The only remaining detail is how to implement saving the relevant hashes in practice. The simplest solution of just saving the hash whenever we create a unit or endorsement has a significant drawback. Imagine we create a unit, save its hash as the hash of the last unit created by us, and then crash before sending the unit to any validator (for any reason, eg. a power outage). In such a case, after restarting, we wouldn't be able to download it again. We wouldn't be able to make sure that we don't equivocate and so we wouldn't start voting again. It is probably the most secure solution, though. An alternative will be discussed in [Rationale and alternatives](#rationale-and-alternatives).

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The consensus component should:

- Whenever a unit or an endorsement is created, emit a request to storage to store the latest data for the era. The data stored would contain:
    - The hash of the latest unit
    - The list of hashes of latest units by each validator endorsed by us

This data should be keyed by the consensus instance ID.

In order to facilitate removing data for old eras (ones that are older than `UNBONDING_PERIOD`), we should also store a map of era ID → instance ID - as instance IDs themselves carry no information about how old the era is. We can, however, identify the era IDs of old eras, and using this map, we could also find the entries to be removed.

Any sending of a unit / an endorsement should only happen after the storage request returns a success response.

- Whenever a new era is started, it should first try to read the entry with latest hashes from storage by submitting a read request.
- If the entry doesn't exist, proceed the same way it is done currently.
- If the entry exists:
    - Pass all the data to the `ActiveValidator` instance.
    - Only create new units if the unit with the latest hash is known to the consensus state.
    - Only create new endorsements if all the previous endorsements are known to the consensus state.

## Drawbacks

[drawbacks]: #drawbacks

Every creation of a unit or an endorsement will now involve storage - however, this seems unavoidable in that we need to store the knowledge of our activity in consensus in some permanent way.

Also, there is a possibility that a node will crash after persisting the hash of a new unit in an era, but before gossiping it - such a situation would render it unable to sync in that era. However, such situations should be unlikely enough to not become a major problem.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

### An alternative hash saving scheme

In order to prevent situations in which we get stuck with a unit hash that no other validator has heard of, we could use a solution like the following:

1. Let's denote n-th unit created by us by U[n].
2. Whenever we create a unit, we _append_ its hash to the list of hashes of the units we created.
3. Whenever we receive a unit citing U[n] from another validators whose weights add up to more than our FTT, we remove the hashes of U[n-1] and older units from the list.
4. When we rejoin an era and try to sync the state, request the latest unit from the list from other validators.
5. If no validator has this unit, remove it from the list and try with the next latest one.

Point 3 makes sure that at least one honest validator has received our unit before we remove its ancestors from our list. This way, we should always have at least one hash that corresponds to a unit received by an honest validator.

Point 5 makes sure that we don't try starting from an older unit unless no validator seems able to send it to us.

However, this scheme still has an important weakness. If only some of the honest validators knew our unit before they all crashed, and the only ones that still have it are malicious, they could withhold this information, prompting us to start at some earlier point in the history and create a conflicting unit. Then they could reveal our old unit and claim that we equivocated.

Because of this risk, we probably shouldn't use this approach and go for the simpler, more secure one.

### Other alternatives

Another simple alternative would be to just store the complete consensus state every time it is modified. This seems excessive, though, and would be much heavier on the IO side, causing a lot of disk activity and potentially slowing down consensus significantly. This approach could probably be optimized a lot by only storing information about changes to the state and then recombining them into the full state when needed, but still, it's probably a lot of unnecessary complexity.

## Prior art

[prior-art]: #prior-art

Permanent storage of state that could be relevant to later executions of the software is a common practice.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- Is the risk of getting stuck with a hash of an unknown unit low enough, or does this possibility require further research?

## Future possibilities

[future-possibilities]: #future-possibilities

None at the moment.
