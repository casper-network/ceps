# Alphanumeric Transfer Id's

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0074](https://github.com/casperlabs/ceps/pull/0074)

In summary, transfer ID's will be extended to accept alphanumeric strings. Currently, transfer IDs are an optional integral type. The goal of this CEP is to update transfer IDs to be a string of alphanumeric characters, an integer or empty.

## Motivation

[motivation]: #motivation

The main motivation for this CEP is to make integration with the Casper Blockchain Network easier for developers and organizations. As an example, if a development or accounting team uses an alphanumeric ID system for their in-house record keeping, they would be able to continue to use the same system, which could make it easier to reference on-chain transactions in their data.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

This CEP introduces a new type that will extend the `id` field of the `Transfer` struct to allow it to store alphanumeric string-based memos: `Memo`.

Alphanumeric strings will contain the characters `0` through `9`, and `a` through `z`. Alphabetic characters can be uppercase or lowercase. Transfer IDs will no longer only be optional integral types, so code that reads or writes transfer IDs will need to be updated.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Here is an example implementation for a new `Memo` type that will be stored in a `Transfer`'s `id` field.

```rust
/// Memo field associated with a [`Transfer`].
#[derive(Debug, Clone, Ord, PartialOrd, Eq, PartialEq, Serialize, Deserialize)]
#[cfg_attr(feature = "json-schema", derive(JsonSchema))]
pub enum Memo {
    None,
    Numeric(u64),
    Alphanumeric(String),
}

impl Memo {
    fn tag(&self) -> u8 {
        match self {
            Memo::None => bytesrepr::OPTION_NONE_TAG,
            Memo::Numeric(_) => bytesrepr::OPTION_SOME_TAG,
            Memo::Alphanumeric(_) => 2,
        }
    }
}

impl Default for Memo {
    fn default() -> Self {
        Memo::None
    }
}

impl ToBytes for Memo {
    fn to_bytes(&self) -> Result<Vec<u8>, bytesrepr::Error> {
        let mut bytes = vec![self.tag()];
        match self {
            Memo::None => (),
            Memo::Numeric(value) => {
                bytes.extend(value.to_bytes()?);
            }
            Memo::Alphanumeric(value) => {
                bytes.extend(value.to_bytes()?);
            }
        }
        Ok(bytes)
    }

    fn serialized_length(&self) -> usize {
        1 + match self {
            Memo::None => 0,
            Memo::Numeric(value) => value.serialized_length(),
            Memo::Alphanumeric(value) => value.serialized_length(),
        }
    }
}

impl FromBytes for Memo {
    fn from_bytes(bytes: &[u8]) -> Result<(Self, &[u8]), bytesrepr::Error> {
        let (tag, remainder) = u8::from_bytes(bytes)?;
        match tag {
            bytesrepr::OPTION_NONE_TAG => Ok((Memo::None, remainder)),
            bytesrepr::OPTION_SOME_TAG => {
                let (value, remainder) = u64::from_bytes(remainder)?;
                Ok((Memo::Numeric(value), remainder))
            }
            2 => {
                let (value, remainder) = String::from_bytes(remainder)?;
                Ok((Memo::Alphanumeric(value), remainder))
            }
            _ => Err(bytesrepr::Error::Formatting),
        }
    }
}
```

`Transfer` would then be changed to hold a non-optional `Memo` instead of the current `Option<u64>`. This relies on the way that we're currently serializing `Option<64>`. The tag for `Option::None` corresponds to `Memo::None` and the `Option::Some` tag corresponds to `Memo::Numeric`. The tag for `Memo::Alphanumeric` is new and will not conflict with the existing tags.

## Drawbacks

[drawbacks]: #drawbacks

Major changes to core features of the blockchain network require significant time and effort to implement, and may also come with unforeseen consequences.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The `Memo` type can be implemented without requiring any data migrations because of the way that the current `Option` tag bytes correspond to the new `Memo` tag bytes. Adding a new, independent text field to `Transaction` would require a production data migration, and it could be confusing to users that are trying to understand the difference between the optional integral type `id` field and the alphanumeric memo field.

## Prior art

[prior-art]: #prior-art

Other blockchain networks allow string based transfer-ids, but it's not exactly clear what advantages or disadvantages that brings. There are also several networks that only allow integral transfer id's, and it's also not clear if there are serious disadvantages to disallowing string based identifiers.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

Is there a practical reason to allow more characters than just those that are alphanumeric?

## Future possibilities

[future-possibilities]: #future-possibilities

This design is open to extension because of the way that the underlying encoding of a `Memo` variant works. The type tag for a `Memo` variant is one byte, since only `0`, `1` and `2`, have been used, there are 253 other variants that could be implemented if need be.
