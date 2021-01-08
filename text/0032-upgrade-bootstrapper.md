# `upgrade-bootstrapper`

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0032](https://github.com/casperlabs/ceps/pull/0032)

The `upgrade-bootstrapper` is a new binary which will be responsible for launching the `casper-node` binaries.  It will
handle transitions between network versions by identifying when a current `casper-node` execution ceases and starting
a new version of the `casper-node` binary.


## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The `upgrade-bootstrapper` will be a relatively simple Rust binary which will run the `casper-node` binary as a child
process.  It will take a single optional argument: the path to the `config.toml` for the `casper-node` binary to be run.

Under normal circumstances, the following steps will be performed:

1. the `upgrade-bootstrapper` is started
2. if the path to the node's `config.toml` was provided via an arg, the `upgrade-bootstrapper` writes the path value to
   a hard-coded, known location (if writing fails, it panics).  Otherwise it reads the path value from the known
   location.  If the path cannot be read from the known location, it panics
3. the `upgrade-bootstrapper` spawns a child process, running the `casper-node` binary in validator mode, passing it the
   path to its `config.toml` as an arg
4. while they are running, before the network reaches the upgrade point, the next version of the `casper-node` gets
   installed, but not run.  The binary is always named `casper-node-next` and its config named `config-next.toml`.  Each
   gets installed to the same location as the current equivalent files.  The location of the `config.toml` can be
   retrieved from the known location used by the `upgrade-bootstrapper` at step 2.  An upgrade file for chainspec
   modifications should also be installed and its path included in the new `config-next.toml`
5. when the current upgrade activation point is reached and the node has finished all its work in that era, it exits
   normally with an exit code 0
6. the `upgrade-bootstrapper` renames
     * `casper-node-next` to `casper-node` (replaces the old binary)
     * `config.toml` to `config-old.toml`
     * `config-next.toml` to `config.toml`
7. the `upgrade-bootstrapper` runs the new `casper-node` in migrate-data mode (see below for further details), passing
   it the path to `config.toml` and `config-old.toml` as args.  The node performs any migration of stored data and
   config options required, then exits normally with an exit code 0
8. repeat from step 3

Should the child process stop with a non-zero exit code at any stage, the `upgrade-bootstrapper` should itself panic.
It is up to the user to restart the `upgrade-bootstrapper` manually.

It is assumed that the `casper-node` binary will live at a known location and that installing an upgraded binary will
place the binary and its config file in a known location.  If we wish to be more flexible about this (particularly the
location of the `casper-node` binary) we can handle these locations in a similar way to the location of the
`casper-node`'s `config.toml`.


## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Changes to existing `casper-node` (running in validator mode)

We require the following changes to `casper-node`:

* running in validator mode, when we reach the upgrade activation point, exit gracefully
* running in validator mode, if the node starts, joins and finds that it has already reached the upgrade activation
  point, exit gracefully
* upgrade activation points should be defined in terms of era IDs, not block heights
* config.toml updated to take the path to the chainspec upgrade file, similar to the `node.chainspec_config_path`
  option.  Alternatively, just remove this option, and require the chainspec, its upgrade file, accounts.csv and the
  config.toml file all to live in the same folder

### Data migration

We require to add a new `migrate-data` subcommand to `casper-node`.  We could provide a separate binary to be used
exclusively for data and config migration, but it seems simpler to just make this a subcommand.  Simpler for the
installer, for the `upgrade-bootstrapper` and simpler to code.

Migration of any of the databases needs to take the following approach by iterating each value in the DB and doing:

1. parse as the old type
2. convert it to the new type
3. store the new type, possibly under a different key
4. if stored under a different key (i.e. it's not just an overwrite), then delete the old value

We need to be able to resume migration in case it's killed while running.  This means we need to ensure that we either
keep track of our progress by writing to disk after each individual item is successfully migrated, or else we ensure
that parsing a new value as an old value fails, or else we ensure that running the conversion step on an
already-converted item is a no-op.

If data migration is going to take a prohibitively long time to execute, we should look to have the `upgrade-installer`
perform _data_ migration only (not config migration) as soon as the installer provides the new `casper-node` binary,
while the current `casper-node` is still running in validator mode.  However, this might result in a doubling or more of
the disk space consumed by the databases, since the migrated data cannot be allowed to overwrite the existing data while
the node is still running as a validator, and _this_ disk consumption might prove to be prohibitive.  Data migration
would also still need to be executed again once the old `casper-node` stops due to reaching the upgrade activation
point, since more data will likely have been written to the databases between then and when the data migration was
previously completed.

### Config migration

Migration will also need to handle converting an old config file to a new one.  For this to work, we need to know which
config values have been modified by the user from their default values.  This requires either default values to be
provided, hard-coded in the node, or else the `migrate-data` subcommand will need to be passed not just the old and new
config.toml files, but also a copy of each with their default values set.

We moved away from hard-coding defaults in the code a while ago, but a challenge to the other approach is ensuring the
user _copies_ the installed vanilla config rather than renaming or deleting it.  Two possible solutions are:
1. have the installer create a read-only copy of the default config exclusively for the `upgrade-bootstrapper`'s use,
   and a separate writable one for the user to modify/copy/move as they wish
1. require that every config value is annotated with a standard comment specifying the default value

It seems like annotating the config file with default values might be somewhat more "normal" and should be fairly simple
to implement and test.

To migrate, the node will identify any modified values in the old config, and apply the same changes to the new config
if the corresponding new config value has not already been modified by the user.


### `upgrade-bootstrapper`

We also need to provide an `upgrade-bootstrapper` binary.  It should live in its own repo outside the `casper-node` one
if setting up CI is not deemed too onerous.  Ideally its tests shouldn't need to run when changes are made to
`casper-node`, and vice-versa.  There also should be no direct dependency between the `update-bootstrapper` and any of
the other casper crates.  Hence it makes sense to keep this separate to the rest of the node project.

The `upgrade-bootstrapper` and `casper-node` must both be capable of being restarted, having been stopped or killed at
any step above, without disrupting this normal flow.  This will require the `upgrade-bootstrapper` to write which step
its about to perform to the hard-coded known location immediately before performing the step.  On starting, the
`upgrade-bootstrapper` will try to read this value and jump to that step.  If the value/file is absent, it assumes it's
a fresh start.


## Drawbacks

[drawbacks]: #drawbacks

Users lose the ability to override config values via command line.


## Unresolved questions

[unresolved-questions]: #unresolved-questions

1. How to get the upgrade activation point agreed by all validators?
    * Could include first activation point in chainspec, and all chainspec upgrade files hold the _next_ activation
      point. (Drawback is that we need to know much further in advance when we want the next activation point to occur)
    * Once new chainspec upgrade file is installed, `upgrade-bootstrapper` could notify the node about its path (via
      IPC or TCP listener on loopback).  Nodes could then use consensus to agree the point (could take a long time to
      get consensus as install times of next version of software could vary widely)
    * On each era change, `casper-node` could read from a file in a known location the next upgrade activation point.
      This could be the chainspec upgrade file, or a standalone one containing just the activation point.

1. Do we have to keep all chainspec upgrade files, or just the latest?  If just the latest, change the chainspec to not
   hold them in a `Vec`.
