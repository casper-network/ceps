# Minimal gas charges

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0069](https://github.com/casperlabs/ceps/pull/0069)

We propose introduction of a minimal gas charge for all deploys, equal to a new, higher gas cost for native transfers.

The minimal charge, added to all generic deploys, is to be 100,000,000 gas units.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

### Chainspec changes

Gas table cost of native transfers to be set to 100,000,000.

### casper-node changes

Execution engine logic to change as follows:

1. Payment phase
    - No changes
2. Execution phase
    - Prior to any other gas charges, add 100,000,000 to gas counter

## Drawbacks

[drawbacks]: #drawbacks

We acknowledge and would like to address concerns about the potential of such a change to disrupt current platform operation and put the platform into a less competitive position relative to comparable L1 platforms.

We expect little disruption of the ecosystem, given adequate warnings to major participants, such as exchanges.

Additionally, given a CSPR price of $0.1, the dollar cost of transfers would still be an order of magnitude lower than the dollar costs of transfers on comparable L1 platforms.

## Alternatives

[alternatives]: #alternatives

No alternatives are known.

## Prior art

[prior-art]: #prior-art

No prior art is known.

## Future possibilities

[future-possibilities]: #future-possibilities

No further development of this feature is expected at present.