# `casper-node-launcher`

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0032](https://github.com/casperlabs/ceps/pull/0032)

The `casper-node-launcher` is a new binary which will be responsible for launching the `casper-node` binaries.  It will
handle transitions between network versions by identifying when a current `casper-node` execution ceases and starting
a new version of the `casper-node` binary.


## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The `casper-node-launcher` will be a relatively simple Rust binary which will run the `casper-node` binary as a child
process.  It will take a single optional argument: the path to the `config.toml` for the `casper-node` binary to be run.

Under normal circumstances, the following steps will be performed:

1. the `casper-node-launcher` is started
2. if the path to the node's `config.toml` was provided via an arg, the `casper-node-launcher` writes the path value to
   a hard-coded, known location (if writing fails, it exits with an error).  Otherwise it reads the path value from the
   known location.  If the path cannot be read from the known location, it exits with an error
3. if in the same folder as the `config.toml` there is a `config-next.toml`, the `casper-node-launcher` runs the
   `casper-node` in migrate-data mode (see below for further details), passing it the path to `config.toml` and
   `config-next.toml` as args.  The node performs any migration of stored data, then exits normally with an exit code 0.
4. the `casper-node-launcher` renames `config-next.toml` if it exists to `config.toml` (replaces the old node config)
5. the `casper-node-launcher` runs the `casper-node` binary in validator mode, passing it the path to its `config.toml`
   as an arg
6. while they are running, before the network reaches the upgrade point, the next version of the `casper-node` gets
   installed, but not automatically run.  The binary is always named `casper-node-next` and its config named
   `config-next.toml`.  Each gets installed to the same location as the current equivalent files.  The location of the
   `config.toml` can be retrieved from the known location used by the `casper-node-launcher` at step 2.  An upgrade file
   for chainspec modifications should also be installed and its path included in the new `config-next.toml`
7. the user, immediately after installing the new software, performs the config migration.
8. when the current upgrade activation point is reached (see below for further details on how the running node will get
   to know of the activation point) and the node has finished all its work in that era, it exits normally with an exit
   code 0
9. the `casper-node-launcher` renames `casper-node-next` to `casper-node` (replaces the old binary)
10. repeat from step 3

Should the child process stop with a non-zero exit code at any stage, the `casper-node-launcher` should itself exit with
an error.  It is up to the user to restart the `casper-node-launcher` manually.

This flow also allows for an installer or user to upgrade the `casper-node-launcher` itself.  The `casper-node-launcher`
should be able to be killed, replaced and restarted while maintaining the integrity of the flow above.

It is assumed that the `casper-node` binary will live at a known location and that installing an upgraded binary will
place the binary and its config file in a known location.  If we wish to be more flexible about this (particularly the
location of the `casper-node` binary) we can handle these locations in a similar way to the location of the
`casper-node`'s `config.toml`.


## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Default locations

The following default locations will be used:

| File                               | Location                                       |
| :--------------------------------- | :--------------------------------------------- |
| `casper-node` binary               | `/var/lib/casper/bin/casper-node`              |
| `casper-node-launcher` binary      | `/var/lib/casper/bin/casper-node-launcher`     |
| `casper-node` config file          | `/etc/casper/config.toml`                      |
| `casper-node-launcher` config file | `/etc/casper/casper-node-launcher-config.toml` |

### Changes to existing `casper-node` (running in validator mode)

As well as the other changes specified in the sections below, we require the following changes to `casper-node`:

* running in validator mode, when we reach the upgrade activation point, exit gracefully.  Specifically, the point at
  which the node should exit is one minute after it has received a third of finality signatures for the switch block.
  The one minute delay is to allow the completion of gossiping messages for the benefit of other peers still awaiting
  the requisite number of finality signatures.  It is an overly simple solution, pending replacement by a more robust
  and efficient one
* running in validator mode, if the node starts, joins and finds that it has already reached the upgrade activation
  point, exit gracefully
* upgrade activation points should be defined in terms of era IDs, not block heights
* config.toml updated to take the path to the chainspec upgrade file, similar to the `node.chainspec_config_path`
  option.  Alternatively, just remove this option, and require the chainspec, its upgrade file, accounts.csv and the
  config.toml file all to live in the same folder
* chainspec loader will need to properly support returning an upgraded chainspec
* initializer reactor will need to support calling the `commit_upgrade` method of contract runtime
* consider creating an empty block immediately after committing an upgrade, in order to capture the global state hash
  after possible data-migration but before the global state has been affected by executing any deploys.

### Learning of a new upgrade activation point

For simplicity's sake, the running `casper-node` will try on every era change to read from a file in a known location
the next upgrade activation point.  This could be the chainspec upgrade file, or a standalone one containing just the
activation point.

### Data migration

We require to add a new `migrate-data` subcommand to `casper-node`.  We could provide a separate binary to be used
exclusively for data and config migration, but it seems simpler to just make this a subcommand.  Simpler for the
installer, for the `casper-node-launcher` and simpler to code.

Currently we only require to migrate global state data.  The data held by the storage component can be left untouched
as it won't be used after a major version protocol upgrade.

Migration of any of the databases needs to take the following approach by iterating each value in the DB and doing:

1. parse as the old type
1. convert it to the new type
1. store the new type, possibly under a different key, or in a new database
1. if stored under a different key (i.e. it's not just an overwrite), then delete the old value

We need to be able to resume migration in case it's killed while running.  This means we need to ensure that we either
keep track of our progress by writing to disk after each individual item is successfully migrated, or else we ensure
that parsing a new value as an old value fails, or else we ensure that running the conversion step on an
already-converted item is a no-op.

Data migration will only be executed once the old `casper-node` stops due to reaching the upgrade activation point, as
there is no time saving to migrating the trie store prior to that point.

### Config migration

We require to add a new `migrate-config` subcommand to `casper-node`.  Config migration will handle converting an old
config file to a new one.  For this to work, we need to know which config values have been modified by the user from
their default values.  This requires either default values to be provided, hard-coded in the node, or else the
`migrate-data` subcommand will need to be passed not just the old and new config.toml files, but also a copy of each
with their default values set.

We moved away from hard-coding defaults in the code a while ago, but a challenge to the other approach is ensuring the
user _copies_ the installed vanilla config rather than renaming or deleting it.  Two possible solutions are:
1. have the installer create a read-only copy of the default config exclusively for the `casper-node-launcher`'s use,
   and a separate writeable one for the user to modify/copy/move as they wish
1. require that every config value is annotated with a standard comment specifying the default value
1. `include_str!` the default config.toml to make it available at compile time

It seems like using `include_str!` might be somewhat more "normal" and should be fairly simple to implement and test.

To migrate, the node will identify any modified values in the old config, and apply the same changes to the new config
if the corresponding new config value has not already been modified by the user.

It is expected that the user will execute this subcommand immediately after a new version is installed, and at least
before it is required by the new `casper-node` binary.

### `casper-node-launcher`

We also need to provide a `casper-node-launcher` binary.  It should live in its own repo outside the `casper-node` one
if setting up CI is not deemed too onerous.  Ideally its tests shouldn't need to run when changes are made to
`casper-node`, and vice-versa.  There also should be no direct dependency between the `casper-node-launcher` and any of
the other casper crates.  Hence it makes sense to keep this separate to the rest of the node project.

The `casper-node-launcher` and `casper-node` must both be capable of being restarted, having been stopped or killed at
any step above, without disrupting this normal flow.

### Fast sync

Fast sync should be unaffected by this proposal, assuming we include certain required fields in Block headers
consistently across all protocol upgrades.  The actual required fields will be specified elsewhere as part of a
separate design proposal.


## Drawbacks

[drawbacks]: #drawbacks

1. We don't have the ability to slash across major protocol version upgrades.  This is not a major problem; only
unsuccessful attempts to disrupt the network will go unpunished.  Successful attempts will be subject to manual slashing
via social consensus.

1. Users lose the ability to override config values via command line.


## Unresolved questions

[unresolved-questions]: #unresolved-questions

1. Do we have to keep all chainspec upgrade files, or just the latest?  If just the latest, change the chainspec to not
   hold them in a `Vec`


## Future possibilities

[future-possibilities]: #future-possibilities

We could aim to have the consensus component reach consensus on newly-read upgrade points, and possibly also on the hash
of the newly-installed casper-node package.

We should also look at auto-upgrades in the future.  This might become the go-to mechanism for upgrades, although that
depends upon the user not needing to manually modify the config file.

We could also consider supporting upgrade points to be associated with block height rather than era ID to allow for more
fine-grained scheduling of upgrades, i.e. supporting upgrades at any point within an era, rather than only at era end.
These would only apply to non-major protocol version changes.
