# SplitSig Recovery Kit

A recovery kit is a portable document that contains everything needed to
re-derive a SplitSig identity key on any device, without the original
browser or application state.

## Background

SplitSig derives an identity key from two pieces:

  identity_privkey = SHA256(lnurl_ecdsa_signature || nonce)

The LNURL-auth signature is deterministic given the same wallet and the
same challenge (k1). The nonce is a random 32-byte value generated once
in the browser and never sent to any server. Neither piece alone is
sufficient to derive the key.

The recovery kit packages the nonce together with the context needed to
reconstruct the correct k1 challenge, so that the same wallet can
reproduce the same signature and therefore the same identity key on any
device.

## Format

A recovery kit is a JSON object with the following fields:
```json
{
  "version": 1,
  "app": "string",
  "auth_domain": "string",
  "context_identifier": "string",
  "linking_pubkey": "string",
  "nonce": "string",
  "created_at": 1234567890
}
```

### Fields

**version** `integer` required

Format version. This document describes version 1. Parsers MUST reject
kits with unknown version numbers.

**app** `string` required

Identifier of the application that created the kit. Reverse domain
notation is recommended. Examples: `trade.trustbro.splitsig`,
`com.example.myapp`. Used for display purposes and to disambiguate kits
from different applications.

**auth_domain** `string` required

The hostname of the LNURL-auth service used during key derivation. This
is the domain component used by LUD-04 wallets to derive the
domain-specific linking key. Example: `signer.trustbro.trade`.

Used during recovery to reconstruct the correct k1 challenge and to
verify that the correct wallet is being used.

**context_identifier** `string` required

Application-defined string that was included in the k1 challenge
derivation. The k1 is computed as:

  k1 = SHA256(context_identifier || nonce_salt)

where `nonce_salt` is an optional application-defined separator. The
exact derivation function is application-defined but MUST be
deterministic and documented by the application.

For applications that use a fixed context (e.g. a service-level
identity), this is a constant string. For applications that scope
identity to a specific resource (e.g. a deal or session), this includes
the resource identifier.

**linking_pubkey** `string` required

The compressed 33-byte secp256k1 public key (hex-encoded) that the
wallet presented during LNURL-auth. Used during recovery to verify that
the correct wallet is being used before attempting key derivation.

**nonce** `string` required

32-byte random value, hex-encoded. This is the secret piece that the
server never holds. Combined with the deterministic LNURL-auth signature,
it produces the identity private key.

**MUST** be generated with a cryptographically secure random number
generator. **MUST NOT** be reused across different identities or
applications.

**created_at** `integer` required

Unix timestamp (seconds) at which the kit was generated. Used for
display purposes only.

## Encoding

Recovery kits SHOULD be encoded as base58check for compactness and error
detection when transmitted as strings. The version byte for the
base58check envelope is application-defined and MUST be documented.

Recovery kits MAY also be stored and transmitted as plain JSON.

## Key Derivation

Given a recovery kit and access to the original LNURL-auth wallet, the
identity key is re-derived as follows:

1. Reconstruct k1 from `context_identifier` using the application's
   documented derivation function.

2. Authenticate with the LNURL-auth service at `auth_domain` using the
   wallet. The wallet's linking key MUST match `linking_pubkey`. If it
   does not, the wrong wallet is being used.

3. Obtain the DER-encoded ECDSA signature from the authentication
   response.

4. Derive the identity key:
```
   identity_privkey = SHA256(ecdsa_signature_bytes || nonce_bytes)
```

5. Derive the identity public key from `identity_privkey` using the
   secp256k1 curve.

## Security Properties

**Server cannot derive the key.** The server sees the ECDSA signature
during LNURL-auth but never receives the nonce. Without the nonce,
SHA256(signature || ?) cannot be computed.

**Browser loss does not lose the key.** The nonce is stored in the
recovery kit, not only in browser storage. Any device with the kit and
the original wallet can re-derive the key.

**Nonce loss loses the key permanently.** If the recovery kit is lost
and the nonce is not stored anywhere else, the identity key cannot be
recovered. Applications SHOULD prompt users to download the kit
immediately after key derivation.

**Kit theft requires the wallet.** Possession of the recovery kit alone
is insufficient. The attacker also needs access to the LNURL-auth wallet
to produce the required signature.

## Security Considerations

- The recovery kit MUST be treated as sensitive material. Anyone with
  both the kit and access to the wallet can derive the identity key.
- Applications SHOULD encrypt the kit at rest where possible.
- The nonce MUST be generated before any authentication takes place, not
  derived from authentication outputs.
- Applications SHOULD display a clear warning that the kit must be saved
  before the user closes the browser or leaves the page.

## Reference Implementation

The SplitSig key derivation scheme and recovery kit format are
implemented in:

- Web signer: https://github.com/Antisys/splitsig
- Escrow application: https://github.com/Antisys/ark-escrow
- NIP proposal: https://github.com/nostr-protocol/nips/pull/2294

## Relation to LNURL Standards

This document is intended to inform a future LNURL Document (LUD)
proposing a canonical recovery kit format for LNURL-auth derived
identities. The LUD would standardise the fields defined here so that
multiple applications can produce interoperable recovery kits.
