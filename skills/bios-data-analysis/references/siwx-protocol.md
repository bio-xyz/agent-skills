# SIWX Protocol — Poll & Artifact Endpoint Authentication

The GET endpoints for data analysis require **SIWX (Sign-In With X)** authentication:
- `GET /api/agents/analysis/tasks/{taskId}` — poll status / retrieve results
- `GET /api/agents/analysis/tasks/{taskId}/artifacts/{artifactId}/download` — download artifacts

Only the wallet that paid for the analysis can access these endpoints. This uses [SIWE (EIP-4361)](https://eips.ethereum.org/EIPS/eip-4361) messages signed with `personal_sign` (EIP-191).

## Challenge-response flow

```
Client                                    Server
  │                                         │
  │  GET /api/agents/analysis/tasks/{taskId} │
  │ ──────────────────────────────────────► │
  │                                         │
  │  401 { error, siwx: SiwxChallenge }     │
  │ ◄────────────────────────────────────── │
  │                                         │
  │  Build SIWE message from challenge      │
  │  Sign with personal_sign (EIP-191)      │
  │  Base64-encode { message, signature }   │
  │                                         │
  │  GET /api/agents/analysis/tasks/{taskId} │
  │  X-SIWX: <base64>                      │
  │ ──────────────────────────────────────► │
  │                                         │
  │  200 { status, answer, artifacts, ... } │
  │ ◄────────────────────────────────────── │
```

Every request goes through this flow. Challenges are single-use (fresh nonce each time).

## SiwxChallenge shape

The 401 response body contains an `siwx` object:

```json
{
  "error": "Authentication required",
  "siwx": {
    "domain": "x402.ai.bio.xyz",
    "uri": "https://x402.ai.bio.xyz/api/agents/analysis/tasks/1e4a1c29-e4d7-4fa7-8c90-ae8f21f27c82",
    "nonce": "random-nonce-string",
    "chainId": 8453,
    "statement": "Sign in to access your BioAgent research results",
    "version": "1",
    "issuedAt": "2026-03-27T12:00:00.000Z",
    "expirationTime": "2026-03-27T12:05:00.000Z"
  }
}
```

Challenges expire after 5 minutes.

## Constructing the SIWE message

Build an [EIP-4361](https://eips.ethereum.org/EIPS/eip-4361) message using the challenge fields plus your wallet address:

```
x402.ai.bio.xyz wants you to sign in with your Ethereum account:
0xYourWalletAddress

Sign in to access your BioAgent research results

URI: https://x402.ai.bio.xyz/api/agents/analysis/tasks/1e4a1c29-e4d7-4fa7-8c90-ae8f21f27c82
Version: 1
Chain ID: 8453
Nonce: random-nonce-string
Issued At: 2026-03-27T12:00:00.000Z
Expiration Time: 2026-03-27T12:05:00.000Z
```

Libraries that construct this for you:
- **TypeScript/Node:** `createSiweMessage()` from `viem/siwe`
- **Python:** `siwe` package or manual string construction (see below)

## Signing and encoding

1. Sign the SIWE message string with `personal_sign` (EIP-191):
   - **viem:** `account.signMessage({ message })`
   - **eth_account:** `Account.sign_message(encode_defunct(text=message))`
2. JSON-encode `{ "message": "<siwe-string>", "signature": "0x..." }`
3. Base64-encode the JSON string
4. Send as `X-SIWX` header

## TypeScript example

```typescript
import { createSiweMessage } from "viem/siwe";

// challenge = the siwx object from the 401 response
const message = createSiweMessage({
  address: account.address,
  domain: challenge.domain,
  uri: challenge.uri,
  nonce: challenge.nonce,
  chainId: challenge.chainId,
  statement: challenge.statement,
  version: challenge.version,
  issuedAt: new Date(challenge.issuedAt),
  expirationTime: new Date(challenge.expirationTime),
});
const signature = await account.signMessage({ message });
const header = Buffer.from(JSON.stringify({ message, signature })).toString("base64");

// Use: fetch(url, { headers: { "X-SIWX": header } })
```

## Python example

```python
from eth_account import Account
from eth_account.messages import encode_defunct
import base64, json

def build_siwe_message(challenge: dict, wallet_address: str) -> str:
    return (
        f"{challenge['domain']} wants you to sign in with your Ethereum account:\n"
        f"{wallet_address}\n"
        f"\n"
        f"{challenge['statement']}\n"
        f"\n"
        f"URI: {challenge['uri']}\n"
        f"Version: {challenge['version']}\n"
        f"Chain ID: {challenge['chainId']}\n"
        f"Nonce: {challenge['nonce']}\n"
        f"Issued At: {challenge['issuedAt']}\n"
        f"Expiration Time: {challenge['expirationTime']}"
    )

def sign_siwx_challenge(challenge: dict, private_key: str) -> str:
    account = Account.from_key(private_key)
    message = build_siwe_message(challenge, account.address)
    signed = account.sign_message(encode_defunct(text=message))
    payload = json.dumps({"message": message, "signature": "0x" + signed.signature.hex()})
    return base64.b64encode(payload.encode()).decode()
```

## Server verification rules

The server checks:
1. `X-SIWX` header present and valid base64 JSON with `message` + `signature`
2. SIWE message has a valid `address` field
3. `domain` in the SIWE message matches the request host
4. Signature is valid (supports EOA via `ecrecover` and smart wallets via EIP-1271)
5. Signer address matches the wallet that originally paid for the analysis task

If any check fails, the server returns 401 (or 400 for malformed headers).

## Wallet requirement

The SIWX signer **must be the same wallet** that paid for the analysis in Step 3. The server stores the paying wallet address on the task record and compares it during verification.
