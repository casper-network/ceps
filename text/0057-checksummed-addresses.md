# Title

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0057](https://github.com/casperlabs/ceps/pull/0057)

This CEP introduces an opt-in checksum scheme to avoid copy errors.
The encoding generalizes [EIP-55][1]. We provide a scheme for
hex-encoded values. The approach encodes a checksum in the
capitalization of hexadecimal digits. While the new format protects
account hashes, it also protects all hex-encoded values.

[1]: https://eips.ethereum.org/EIPS/eip-55

## Motivation

[motivation]: #motivation

Copy errors when entering account addresses may result in the loss of
millions. A checksum scheme avoids this by including redundant
information.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The scheme encodes a checksum in the letter capitalization of hex-encodings.
It extends [EIP-55][1] to work with arbitrary length arrays.

The encoding uses the hash of the input to determine capitalization.
The hash is broken into 4-bit chunks called *nibbles*. A nibble
corresponds to a hexadecimal digit. The procedure also breaks the
input into nibbles. Each input nibble uses the hash nibble with the
same index for capitalization. If the corresponding hash nibble is
less than or equal to 7, the hex character is capitalized. Otherwise,
it is lower case.

Decoding verifies the correct capitalization of each letter. The
decoder skips verification if all letters are lower case or upper
case.

[1]: https://eips.ethereum.org/EIPS/eip-55

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The design leverages two special `encode` and `decode` function which
work on hexadecimal strings.

### Converting Bytes to Nibbles

The encoding procedure relies a helper function which converts bytes
to nibbles. As mentioned in the [guide-level explanation](#guide-level-explanation), 
nibbles are 4-bit chunks. They correspond to hexadecimal digits.

```rust
fn bytes_to_nibbles(bytes: &[u8]) -> Vec<u8> {
    let mut output_nibbles = Vec::with_capacity(bytes.len() * 2);
    for b in bytes {
        output_nibbles.push(b >> 4);
        output_nibbles.push(b & 0x0f);
    }
    output_nibbles
}
```

### Encoding

The `encode` function below takes a byte array as input and outputs a
check-summed hex string. See the [guide-level
explanation](#guide-level-explanation) for a description of the
algorithm.

```rust
const HEX_CHARS: [char; 16] = [
  '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f',
];

pub fn encode(input: &[u8]) -> String {
  let input_nibbles = bytes_to_nibbles(input);
  let hash_nibbles = bytes_to_nibbles(&blake2b_hash(input));
  let hash_nibble_count = hash_nibbles.len();
  let mut hex_output_string = String::with_capacity(input_nibbles.len());
  for idx in 0..input_nibbles.len() {
    let c = HEX_CHARS[input_nibbles[idx] as usize];
    if hash_nibbles[idx % hash_nibble_count] <= 7 {
      hex_output_string.extend(c.to_uppercase())
    } else {
      hex_output_string.extend(c.to_lowercase())
    }
  }
  hex_output_string
}
```

### Decoding

The `decode` function is presented below. As discussed, if the input
is all upper-case letters or all lower-case letters verification is
skipped.

```rust
fn string_is_uppercase(input_string: &str) -> bool {
    input_string
        .chars()
        .filter(|c| c.is_alphabetic())
        .all(|c| c.is_uppercase())
}

fn string_is_lowercase(input_string: &str) -> bool {
    input_string
        .chars()
        .filter(|c| c.is_alphabetic())
        .all(|c| c.is_lowercase())
}

pub fn decode(input_string: &str) -> Result<Vec<u8>, base16::DecodeError> {
    let bytes = base16::decode(input_string)?;
    if !(string_is_uppercase(input_string) || string_is_lowercase(input_string)) {
        let checksum_hex_bytes = encode(&bytes).into_bytes();
        let input_string_bytes = input_string.as_bytes();
        for idx in 0..input_string_bytes.len() {
            if checksum_hex_bytes[idx] != input_string_bytes[idx] {
                return Err(base16::DecodeError::InvalidByte {
                    index: idx,
                    byte: input_string_bytes[idx],
                });
            }
        }
    }
    Ok(bytes)
}
```

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

An alternative would be to include (optional) checksum fields. That
solution and the one presented are not mutually exclusive. Alternating
capitalization operates at the wire message level. Checksum fields are
at the data level. A careful review is necessary to determine where
optional checksum fields are appropriate.

## Prior art

[prior-art]: #prior-art

As discussed, this work generalizes [EIP-55][1].

[1]: https://eips.ethereum.org/EIPS/eip-55

## Unresolved questions

[unresolved-questions]: #unresolved-questions

The encoding works best for APIs that use JSON with certain fields
hex-encoded. However, an API might encode a lot of data as hex. In
that a case it will not protect from user error.