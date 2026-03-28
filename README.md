# SplitSig

Non-custodial key derivation from Lightning wallet authentication.

```
privkey = SHA256( ecdsa_signature + nonce )
                  ───────────────   ─────
                  server sees this   browser-only
```

The server sees the ECDSA signature (inherent to LNURL-auth). The browser generates a random nonce the server never sees. Neither piece alone can derive the private key.

## Install

```bash
npm install github:Antisys/splitsig
```

```javascript
import { generateNonce, deriveKey, verify } from '@antisys/splitsig';

const nonce = generateNonce();
const key = deriveKey(signature, nonce);
// key.privateKey     — stays in browser
// key.publicKeyXOnly — send to server
```

## Demo

[Watch the full flow (video)](demo.mp4)

An interactive demo of the SplitSig key derivation protocol:

1. **Generate Nonce** — browser creates a random 256-bit secret
2. **LNURL-auth** — wallet signs a challenge, signature passes through server
3. **Derive Key** — browser computes `SHA256(sig + nonce)`, server can't
4. **Register Pubkey** — only the public key is sent to the server
5. **Recovery Kit** — nonce for key re-derivation on any device with the same wallet

The LNURL-auth linking key is derived from the wallet's BIP-32 seed (LUD-04). It is stable across app updates, OS upgrades, and device changes — as long as the wallet seed is the same, the derived key is reproducible.

## Usage

Open `index.html` in a browser. No build step needed.

```bash
python3 -m http.server 8080
```

## Why This Matters

Web applications that need signing keys face the "trusted code delivery problem." Every existing approach sacrifices something:

- **Server-generated keys**: custodial
- **Browser-generated keys**: lose the backup, lose the funds
- **Signature-derived keys** (zkSync, Umbra): server sees the signature, server has the key

SplitSig adds a second factor (the nonce) that the server never sees. Even if the server stores every signature, it cannot derive any user's private key.

## Prior Art

| System | Key from signature? | Split knowledge? |
|--------|-------------------|-----------------|
| zkSync | yes | no |
| Umbra Protocol | yes | no |
| EIP-2645/StarkWare | yes | no |
| Web3Auth/tKey | no (Shamir split) | yes |
| **SplitSig** | **yes** | **yes** |

To our knowledge, this is the first implementation combining signature-derived keys with client-side nonce splitting.

## Security Model

- **Compromised server**: has signature, not nonce. Cannot derive key.
- **Compromised browser**: has both pieces. Can derive key. (Inherent to all web apps.)
- **Lost recovery kit**: key cannot be re-derived. Application must provide a fallback.
- **Stolen recovery kit**: has nonce, not signature. Cannot derive key without the wallet.
- **Wallet seed lost**: different seed = different signature = different key. Same as any seed-based system.

## Applications

Any web application using LNURL-auth that needs non-custodial signing keys: Nostr clients, ecash wallets, swap services, escrow, multisig coordinators.

See [PROTOCOL.md](PROTOCOL.md) for the full specification.

## License

GPL-3.0
