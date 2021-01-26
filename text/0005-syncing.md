# Node syncing/joining

## Introduction ##

The purpose of this document is to propose a process for a node joining the existing network, download the missing state and start participating in the protocol. Lay out the components that participate in the process, what functionalities they need and how do they interact with one another.

The syncing process has to have three properties, it has to be:

1. **secure** – download fork that honest validators are building on.
2. **correct** – only correct vertices of the DAG must be added to the local state; once validator starts creating new vertices they mustn't be invalid.
3. **fast** – download only what's necessary.

#### Security ####

Before we start validating we need to make sure that we're downloading the correct version of the history. Since a new validator (node) hasn't been observing the chain from the beginning (Genesis) it cannot unequivocally tell which chain is correct (weak subjectivity) – it needs to trust someone/something.  

#### Correctness ####

Node needs a way of verifying the correctness of messages it's receiving. Some correctness conditions are local (signature, invalid hash, invalid message structure) but some are global (era-wise) and depend on previous messages.

#### Speed ####

When optimizing for speed of the syncing process we need to make sure that we don't download data that doesn't need downloading and try downloading data from multiple sources since dowloading a single vertex should take more time than adding it to the state (i.e. download vertices in parallel).

---------------------------------

### Additional requirements ###

There are also additional, functional requirements from an active node (note that I mean *node* and not *validator*):

1. It should gossip only valid messages.
2. It should gossip messages from eras it's not bonded in.

To achieve **1.** we need a knowledge about the whole DAG – otherwise we cannot validate messages. Receiver of an invalid message should punish its creator and also the sender. This means that if we are an active node in the current era and we are expected to gossip current-era messages, we need that era's whole DAG. Even if we are not a ***validator*** in the current era we need to receive and process every vertex of that era.

Point **2.** needs a bit more context: our consensus is using eras to rotate validator set. Only validators bonded in the current era are allowed to produce messages in that era. Once a new era is started, there's an era tail – time when validators produce more messages in that era even though there's a new era running. Validator can equivocate in the past era, and while we have its stake in the PoS contract (until the unbonding period passes), we have to punish him for that (take its staked tokens). Slashing an equivocating validator needs a cryptographic proof – a pair of messages created by the validator that prove an equivocation. 

With this context in mind, we arrive at the conclusion that in order for current-era validators to detect equivocations in the past eras, they need to observe the DAG of previous eras (not yet unbonded ones) – accept vertices created in past eras.

Given all the above:

> Active node needs to know whole DAGs of all active (bonded) eras to correctly participate in the protocol.

## How to synchronize ##

### Building up the initial (Global) state ###

Before we start accepting consensus messages we need to learn what are the current era validators – this information lives in the Global State. We also need the Global State if we want to execute deploys. 

A node is started with an initial hash of the linear chain block (hash of the finalized block) that it gets from a trusted source (a friend, blockchain explorer, CasperLabs organization, newspaper). This block needs to be younger than the unbonding period (era in which it was created is still active). Otherwise, node is not guaranteed to receive consensus messages that will allow it advance the chain. Also, we cannot trust messages created in already inactive (unbonded) eras.

#### Prerequisites ####

1. Add `era_id` to `BlockHeader`
2. Add `switch_block` flag to `BlockHeader`
3. (optional) Add next-era validator set to the `BlockHeader` if`block.switch_block = true`.

Normal way of operation of an active node is that Consensus component will send `FinalizedBlock` messages to the `BlockExecutor`. `FinalizedBlock` message has fields like:

* `switch_block` flag,
* `random_bit`
* `era_id` (doesn't yet have but it should)
* `Vec<SystemTransaction>`

#### Variant A ####

Once we receive the finalized block we started the node with, we check if we have all of its dependencies (parent block, deploys) and if not we ask peers for them. This process is repeated recursively until we have downloaded all of the dependencies and we can start executing deploys from the linear chain (starting from the oldest one).

**Necessary work:**

1. Start a node with the trusted hash.
2. Ask peers for the initial block from the linear chain.
3. Synchronize all missing dependencies.
4. Execute deploys from the linear chain in the order of them appearing in the linear chain.

`BlockExecutor` sees whether it's a `switch_block` and whether it should run an auction for era `era_id + AUCTION_DELAY` . As a result of execution, it will produce a `Block` instance (that is a block of the linear chain) but also an event `NextEraValidators(finalized_block.era_id + 1, Set<Validator>)` that informs Consensus about the next era validators. Consensus component will use new validators (and random bits from blocks between booking block and key block of the next era) and create an instance of new era.

Same mechanism can be used in a node that is catching up – it will use `Block`s from the linear chain to build up its Global State via `BlockExecutor` (we need a mechanism of turning `Block` into `FinalizedBlock` as that's the input that `BlockExecutor` understands) and create new eras. 

Obviously, over the course of synchronization some eras will become obsolete but the assumption is that eras are deactivated/removed by a separate mechanism. For example, `BlockExecutor` component already knows about the unbonding delay so it could send an event to a consensus component that it should deactivate an era (or if consensus component needs this knowledge it could deactivate old eras by itself).

#### Variant B ####

Since downloading the whole linear chain (and most importantly, the deploys) is a time-consuming operation, a faster alternative would be to just sync the Global State at the initial hash. Global State is a merkle trie so it's easily splittable and has an easy way of verifying its correctness (compare root hashes).

If we choose to synchronize Global State directly though, the above won't work. We need to be able to create active eras just by looking at the Global State. This should also be fairly trivial since PoS contract can track past eras, current ones and future (auctions already executed but era not yet started). We can use information from the cache and create instances of eras that are still active.

**Necessary work:**

1. Start a node with the trusted hash
2. Ask peers for the initial block from the linear chain.
3. Synchronize Global State at the post-state hash of the initial block.
4. Validate correctness of the Global State with the post-state hash from the initial block.


---------------------
Once whole linear chain is "consumed" and state is built-up, we need to synchronize DAG(s) of still active eras.

### Synchronizing DAG(s) ###

#### Variant A (quick-n-dirty) ####

Once node has built up its Global State and has necessary eras started it could rely on already existing mechanism of syncing nodes – i.e. it could transition into an *active* node (not a validator) and wait for new messages. Upon receiving a message for which it doesn't have all the necessary dependencies it will already start synchronizing.

#### Variant B

If we want to fully sync the DAG(s) before transitioning from the synchronizing to an active node modes we need to ask peers for their latest DAG tips and compare it to our own view. Requires:

* adding a new variant to `HighwayMessage::GetTips` which should return node's `panorama`
* joining node should send `GetTips` message to its peers for all active eras
* once `panorama`(s) are received, sender should then start requesting for dependencies in a depth-first manner 
* when syncing `panorama`(s) is done, transition to an active node


