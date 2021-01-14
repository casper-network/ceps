# Safe Script for Managing Signing Keys Vs. Withdraw Keys for Validators

## Summary

[summary]: #summary

Create a safe script for managing keys for validators.

## Motivation

[motivation]: #motivation

Some instances of running a validator node requires either sharing of keys to employees, partners, co-workers, etc.. And it would be safe practice to allow signing keys to be shared, while transfer keys are kept safely offline in more restricted custody. Not only would it be safer to keep these keys offline, but it would allow mitigation of damages in the case of a hack, or internal problem with shared custody of master keys that are capable of signing AND transfering. However, a "safe" and well-tested script would be desirable because of potential risks of accidentally revoking transfer/signing powers to the wrong key set, or syntax errors that would lock out anyone/everyone from using that set of keys for its intended purpose.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

A CLI interface that would require specification of keys and corresponding authorizations would be ideal like the following from a default setup:

```
$ validator-key-management --help
 Validator Key Management

 usage:
 	assign-auth-transfer [OPTIONS...]
 	assign-auth-signing [OPTIONS...]
 	revoke-auth-transfer [OPTIONS...]
 	revoke-auth-signing [OPTIONS...]
 	list-keys

 options:
 	--public-key-hex
 	--public-key-path
 	--secret-key-path
 	--master-secret-key


$ validator-key-management assign-auth-transfer \
	--public-key-hex "/path/to/new/public_hex" \
	--public-key-path "/path/to/new/public_key.pem" \
	--secret-key-path "/path/to/new/sec_key.pem" \
	--master-secret-key "/temp/path/to/master/sec_key.pem"

$ SUCCESS - <::public_hex::> - AUTHORIZED FOR: TRANSFER


$ validator-key-management assign-auth-signing \
	--public-key-hex "/path/to/new/public_hex" \
	--public-key-path "/path/to/new/public_key.pem" \
	--secret-key-path "/path/to/new/sec_key.pem" \
	--master-secret-key "/temp/path/to/master/sec_key.pem"

$ SUCCESS - <::public_hex::> - AUTHORIZED FOR: SIGNING


$ validator-key-management revoke-auth-transfer \
	--public-key-hex <::public_hex::> \
	--master-secret-key "/temp/path/to/master/sec_key.pem"

$ SUCCESS - <::public_hex::> - REVOKE FOR: TRANSFER


$ validator-key-management revoke-auth-signing \
	--public-key-hex <::public_hex::> \
	--master-secret-key "/temp/path/to/master/sec_key.pem"

$ SUCCESS - <::public_hex::> - REVOKE FOR: SIGNING


$ validator-key-management list-keys

 1. (master)
 	PUBLIC_KEY:			011117189c666f81c5160cd61...
 	SECRET_KEY:			unavailable
 	AUTHORIZATIONS
 		- SIGNING
 		- TRANSFER

 2.
 	PUBLIC_KEY:			011731928e019874baf13488d...
 	SECRET_KEY:			"/etc/casper/validator_keys/secret_key.pem"
 	AUTHORIZATIONS
 		- SIGNING

```

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## Drawbacks

[drawbacks]: #drawbacks


## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

"This can be done without extending current key management model. It is possible to provide a different wallet for seniorage/fees than the one tied to the validator."
Mateusz, core dev

I wouldn't doubt there could be several ways to achieve the goals written in the summary. An easy method would be best regardless so as not to risk syntax errors and producing a lockout.

## Prior art

[prior-art]: #prior-art


## Unresolved questions

[unresolved-questions]: #unresolved-questions


## Future possibilities

[future-possibilities]: #future-possibilities

A possibly GUI to emulate the CLI interface for this intended functionality would be great in the future.
