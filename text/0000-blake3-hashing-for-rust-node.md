# Replace use of blake2 hashing with blake3

https://github.com/CasperLabs/ceps/pull/8

## Summary

[summary]: #summary

Update project dependencies to include Blake3 hashing, reported to be much faster.

## Motivation

[motivation]: #motivation

It could reasonable to implement this as a project dependency replacing blake2b hashing because it could result in faster internal hashing, and thus a faster network.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Since the Rust version of the node software is already under development, this wouldn't produce any noticeable difference for end users at first glance. Seeing as how Blake3 hashes are identical to Blake2b hashes, 256bit, are compatible with the same 64bit architecture, and are highly parallelizable, this would be totally an internal update. One would expect to notice an overall increase in speed after the update.

Rust implementation found at:
https://crates.io/crates/blake3 and https://docs.rs/blake3/0.3.6/blake3/

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

According to the creators, a test was performed on an AWS c5.metal, 16KiB input, 1 thread, producing the following results compared one against the other:

	Blake2b - 1312 MiB/s
	Blake3  - 6866 MiB/s

These test results were pulled straight from the Blake3 team's repo. Reported to be performed on an Intel Cascade Lake-SP 8275CL modern processor.

## Drawbacks

[drawbacks]: #drawbacks

Why should we *not* do this?
Could break connected logic if the Bouncycastle library is too heavily relied upon. However it may not be an issue in the new Rust codebase if different dependencies are present.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Rationale speaks for itself I believe, anything that could speed up the network without causing a problem or risks creating a new vector of attack is a positive update.

## Prior art

[prior-art]: #prior-art


## Unresolved questions

[unresolved-questions]: #unresolved-questions


## Future possibilities

[future-possibilities]: #future-possibilities

The reason this issue has been brought up is because of the recent Rust re-write of the node software, as Blake3 has been primarily implemented in Rust by the creators, assuming it would be ideal to address before the first release.
