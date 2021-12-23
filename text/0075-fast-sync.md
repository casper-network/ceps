# Fast Synchronization

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0000](https://github.com/casperlabs/ceps/pull/0000)

Joining nodes avoid executing every block by downloading the database
instead. This is *fast syncing*.

## Motivation

[motivation]: #motivation

For a long chain, it is faster to download the database than execute
every block. The chain's database contains account balances. It also
contains the smart contracts and their state.

Nodes only need recent database states to operate, so it saves disk
space to fast sync.

Fast syncing avoids the need to be backward compatible with old
versions. This is because the current executable may not support
executing old blocks. New versions need to support old executables
downloading blocks.

After implementation nodes will join the network by fast syncing.
Operators will only need the current to join the network.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Fast syncing nodes use a trusted hash to download the database in a
secure manner. The trusted hash enables secure download of the latest
block header. That header contains the hash of the database state. Using
that database hash, nodes download the database state in a
BitTorent-like manner. After that nodes execute blocks. When they have
caught up to the current *era* they run the consensus algorithm.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The following `rust` pseudo-code captures the fast-sync logic:

```rust
pub async fn fast_sync(trusted_hash: Digest, chainspec: &Chainspec) -> BlockHeader {
    let trusted_header: BlockHeader = fetch_header_by_hash(trusted_hash).await;

    // Get the initial validators
    let mut parent_hash: Digest = trusted_header.parent_hash;
    let mut validators: BTreeMap<PubKey, Weight> = loop {
        let header: BlockHeader = fetch_header_by_hash(parent_hash).await;
        if let Some(validators) = header.next_era_validator_weights() {
            break validators;
        }
        parent_hash = header.parent_hash;
    };

    // Get the current header
    let mut current_header: BlockHeader = trusted_header;
    while !is_current_era(current_header) {
        let next_height: u64 = current_header.height + 1;
        if let Some(next_header) = fetch_header_by_height(validators, next_height).await {
            current_header = next_header;
        }
        if let Some(new_validators) = current_header.next_era_validators_weights() {
            validators = new_validators;
        }
    }

    // Get old blocks so we have deploy hashes to avoid replay attacks
    let mut walkback_header: BlockHeader = current_header.clone();
    while walkback_header.timestamp + chainspec.deploy_config.max_ttl > Timestamp::now() {
        walkback_header = fetch_block_by_hash(walkback_header.parent_hash)
            .await
            .header;
    }

    // Consensus needs the validators of the last couple of eras to initialize
    // Get the old block headers so we have the validators from the switch-blocks
    let earliest_era_needed_by_consensus: u64 = current_header.era_id - 2;
    while walkback_header.era_id > earliest_era_needed_by_consensus {
        walkback_header = fetch_header_by_hash(walkback_header.parent_hash).await;
    }

    // Download the block chain database (aka the trie-store)
    let mut outstanding_trie_hashes: Vec<Digest> = vec![current_header.state_root];
    while let Some(trie_hash) = outstanding_trie_hashes.pop() {
        let trie: Trie<Key, StoredValue> = fetch_trie(trie_hash).await;
        let new_outstanding_trie_hashes =
            put_trie_and_find_missing_descendant_trie_hashes(trie).await;
        outstanding_trie_hashes.extend(new_outstanding_trie_hashes);
    }

    // Download and execute blocks to current
    while !is_current_era(current_header) {
        let next_height: u64 = current_header.height() + 1;
        if let Some(next_block) = fetch_block_by_height(validators, next_height).await {
            // Fetch the deploys and transfers
            let mut deploys: Vec<Deploy> = vec![];
            for deploy_hash in next_block.deploy_hashes() {
                deploys.push(fetch_deploy(effect_builder, *deploy_hash).await);
            }
            let mut transfers: Vec<Deploy> = vec![];
            for transfer_hash in next_block.transfer_hashes() {
                transfers.push(fetch_deploy(effect_builder, *transfer_hash).await);
            }

            // Execute the block
            execute_block(current_header.state_root, new_block, deploys, transfers).await;
            current_header = new_block.header;
        }
        if let Some(new_validators) = current_header.next_era_validators() {
            validators = new_validators;
        }
    }

    current_header
}  
```

The returned header is then used by the node to join consensus.

Note that deploys and `Trie<Key, StoredValue>` may be downloaded in
parallel as an optimization.

## Drawbacks

[drawbacks]: #drawbacks

Executing blocks may be faster than downloading the database. This is
especially true for implementations that are not optimized.

## Prior art

[prior-art]: #prior-art

-  [go-ethereum][1] has a sophisticated fast sync implementation, similar to the one
   described here. It downloads both the database *and* executes blocks simultaneously.
-  Tezo's [*snapshots*][2] provide similar functionality to what is described here.

[1]: https://geth.ethereum.org/docs/faq#:~:text=Q.%20How%20do%20Ethereum%20syncing%20work%3F
[2]: https://blog.nomadic-labs.com/introducing-snapshots-and-history-modes-for-the-tezos-node.html#:~:text=in%20the%C2%A0future.-,Snapshots,-As%20the%20chain

## Future possibilities

[future-possibilities]: #future-possibilities

Optimizations following `go-ethereum` are desirable.

Fast syncing enables disk recovery. In particular old blocks can be
deleted. These blocks are not needed to join the network.