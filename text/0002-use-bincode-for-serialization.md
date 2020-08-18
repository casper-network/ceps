# Use Bincode for serialization

## Summary

[summary]: #summary

RFC PR: [casperlabs/rfcs#0002](https://github.com/casperlabs/rfcs/pull/0002)

Currently there are multiple serialization formats in use inside the Rust node's storage and various networking protocols. This proposal outlines why they can (and should) all be replaced with [Bincode](https://crates.io/crates/bincode).

## Motivation

[motivation]: #motivation

Serialization is a core necessity for the CasperLabs node, as it has to transfer large amounts of data safely across the network and store it. Any overhead results in reduced performance, while any security issues will instantly expose the node to attackers, as they are on a public network.

Bincode is one of the most efficient Rust-native serialization formats available. Until recently it was unclear whether it held up in terms of security, but since these questions seem to have been [answered](https://github.com/servo/bincode/pull/346), it is likely the best available candidate today.

As a result, serialization and deserialization should become a non-issue for any Rust-only part of the software, as it can leverage the [serde](https://crates.io/crates/serde) and [bincode](https://crates.io/crates/bincode) crates to great effect.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Two core problems that need to be solved through serialization are storage and networking: How are we going to persist data on disk, and how are we going to send it across the network. Storage typically requires a very efficient binary representation and invariance across multiple versions of the software, but can rely on trusting its saved data. Networking, while also sensitive to serialized size, most importantly requires safety from malicious inputs.

The [Bincode](https://crates.io/crates/bincode) format used to have no answers for these questions before, but recently they were [addressed in issue #345 of the project](https://github.com/servo/bincode/issues/345#issuecomment-673025338). Given a correct configuration, the format should be:

* Invariant over endianness
* Stable across minor versions
* Secure against untrusted inputs (no undefined bevavior)
* Protecting against memory-exhaustion (provided it is configured correctly)
* Space efficient (with varint encoding enabled)

**Varint encoding**: Variable length integer encoding encodes integers in different sizes depending on their size. For example, a `u32` might take up 1-5 bytes depending on its values, as opposed to the usual 4 bytes by encoding small values with a single byte.

Some alternatives considered directly were JSON, Msgpack, CBOR; see the alternatives section.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The code changes are minimal due to the fact that the serialization method is already behind a layer of abstraction in the form of the `serde::Deserialize` and `serde::Serialize` traits.

Changing these typically involves only switching out a single method call, e.g. `rmp_serde::write` with `bincode::serialize_into`.

### Encoding module

To make this even more seamless, the choice of encoding and setup will be hidden behind an encoding module that prevents an API for various parts of the node to encode things, making the algorithm used an implementation detail.

## Drawbacks

[drawbacks]: #drawbacks

There are some drawbacks with this approach that stem not necessarily from `bincode` itself, but the approach to serialization as a whole.

No versioning is applied to the serialization at all right now, and this is not addressed in this RFC. Changes in protocol, either by changing the data structures or the serialization method, will require some sort of version tag on the format. This should usually be addressed by the node during connection setup.

The same holds true for stored data; a single byte versioning tag should be sufficient to side-step these problems. It is worth mentioning that some other serialization formats like [ProtoBuf](https://developers.google.com/protocol-buffers) have this functionality built in, but we currently see very little use in backwards compatibility, as there is no incentive in running or supporting an outdated node on the network.

For storage, a versioning tag should be introduced, possibly early, as nodes will want to transfer their stored state to avoid large redownloads. This is not addressed here as well though.

There is a drawback to varint encoding, it is rather branch-heavy, causing more CPU use (a branch here is a decision point where the CPU may take a different path of execution that anticipated, causing it to lose already predicted results or cache hits). <https://stackoverflow.com/a/24642169> has a bit more information about this.

The biggest drawback of Bincode itself is that there is no spec *yet*. Having a fully specced out format is a big advantage when it comes to safety in terms of being able to rely on it long-term.

Not having a spec makes it harder to reimplement Bincode in other languages, there are almost zero implementations outside of Rust. This is not an issue for storage, but could be a problem for interaction with node implementations that are not written in Rust. Since they are not on the horizon today this is not considered an issue at this time. If the requirements should change to include these, allowing a switch of the used encoding protocol should be a straightforward.

An unlikely but potentially painful occurence would be a major version change in the Bincode crate, as there is currently no versioning built into our serialization code. Every binary published should be sure to use a proper `Cargo.lock` to prevent this from happening.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Some alternatives were considered but eliminated:

### Language independent formats

There are formats available like [ProtoBuf](https://developers.google.com/protocol-buffers), [Cap'n Proto](https://capnproto.org/) or [Flatbuffers](https://google.github.io/flatbuffers/), which in general are developed by large tech companies for either internal messaging with maximum performance or externally facing interfaces.

All of them require an extra step during compilation, making the build process more complicated then it needs to be. If external interfaces are required for which JSON is too slow, these should be revisited. Serde-based serialization is chosen over these for simplicity otherwise.

### msgpack

Msgpack is a binary JSON-like protocol, encoding key-value pairs in a tightly packed form. It is mature and has time-tested spec. Beware that there are at least two crates with an implementation, the better maintained one seems to be the [rmp](https://crates.io/crates/rmp)-family of crates.

The `rmp` crates feature a "compact" encoding where all fields of a struct are encoded as an array without field names. This is a non-standard feature, which results in no struct names being stored, causing the size of the output to shrink dramatically.

### CBOR

CBOR is a [somewhat hostile](https://github.com/msgpack/msgpack/issues/129) fork of msgpack, it is expected to be similar to msgpack in most aspects. The Rust library does not offer support for serializing date and time values, while `rmp` does, thus it was rejected.

### Benchmarks

A benchmarking tool was written, solely for examining the message size for representative sample data, available at [casperlabs/serialization-shootout](https://github.com/CasperLabs/serialization-shootout/). Here are the results:

| Data                        | msgpack         | msgpack (compact) | bincode         | bincode (varint) |
| :-------------------------- | :-------------- | :---------------- | :-------------- | :--------------- |
| single deploy               | 771 (1.55)      | 623 (1.25)        | 499 (1.00)      | 417 (0.84)       |
| 10 deploys                  | 7556 (1.51)     | 6076 (1.22)       | 4992 (1.00)     | 4177 (0.84)      |
| 500 bytes Vec (naive)       | 748 (1.47)      | 748 (1.47)        | 508 (1.00)      | 503 (0.99)       |
| 500 bytes Vec (serde_bytes) | 503 (0.99)      | 503 (0.99)        | 508 (1.00)      | 503 (0.99)       |
| 12 megs (naive)             | 18874968 (1.50) | 18874968 (1.50)   | 12582920 (1.00) | 12582917 (1.00)  |
| 12 megs (serde_bytes)       | 12582917 (1.00) | 12582917 (1.00)   | 12582920 (1.00) | 12582917 (1.00)  |
| ballot vertex               | 513 (1.55)      | 439 (1.33)        | 330 (1.00)      | 277 (0.84)       |

Note that the naked `Vec` benchmarks are somewhat unrealistic im real-world situations except when very large deploys are involved.

`bincode` with default settings is used as a reference here, the last column shows it with variable integer length encoding enabled. Msgpack always has varint encoding enabled as part of the spec.

Note that CPU time was _not_ measured, as we expect data transfer amounts to be a bigger bottleneck than deserialization of data. Profiling a more mature node a few month down the line may confirm or shatter this assessment.

In terms of size, Bincode with varint encoding is a clear winner, showing a 25-50% advantage in size in all but naked bytes serialization.

## Prior art

[prior-art]: #prior-art

All the protocols listed above. Most other projects reviewed seem to use one of the formats mentioned in this document.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

None at this time.

## Future possibilities

[future-possibilities]: #future-possibilities

**Compression** is not considered in this document. Any reasonable compression algorithm (e.g. GZIP) will make a large part of the repetitive fields used by Msgpack a moot point. Additionally, it will work well on non-variable length encoded integers. It is possible that different settings or serialization mechanisms could improve performance, so these should be revisited in the accompanying RFCs.

**Profiling** should be done a few months down the line, regardless of the protocol chosen, to ensure the serialization layer is not a bottleneck.
