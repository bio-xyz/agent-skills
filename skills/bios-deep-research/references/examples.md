# Examples & Dependencies

## Minimal curl examples

### API key path

```bash
# 1. Start research
curl -sS -X POST https://api.ai.bio.xyz/deep-research/start \
  -H "Authorization: Bearer $BIOS_API_KEY" \
  --data-urlencode "message=What is NAD+?" \
  --data-urlencode "researchMode=steering"
# → 200 with conversationId

# 2. Poll
curl -sS "https://api.ai.bio.xyz/deep-research/{conversationId}" \
  -H "Authorization: Bearer $BIOS_API_KEY"

# 3. Follow-up (steering only)
curl -sS -X POST https://api.ai.bio.xyz/deep-research/start \
  -H "Authorization: Bearer $BIOS_API_KEY" \
  --data-urlencode "message=Your follow-up question" \
  --data-urlencode "conversationId=CONVERSATION_ID" \
  --data-urlencode "researchMode=steering"

# 4. List past sessions
curl -sS "https://api.ai.bio.xyz/deep-research?limit=20" \
  -H "Authorization: Bearer $BIOS_API_KEY"
```

Note: the direct BIOS API expects **form data**, not JSON. Always use `--data-urlencode` for user-supplied input.

### x402 path (manual signing)

```bash
# 1. Get payment requirements (x402 v2 in response body)
curl -s -X POST https://x402.ai.bio.xyz/api/deep-research/start \
  -H "Content-Type: application/json" \
  -d '{"message":"What is NAD+?","researchMode":"steering"}'
# → 402 with x402Version:2 payment requirements in JSON body

# 2. Sign the authorization (language-specific, see SKILL.md Step 2)

# 3. Submit with PAYMENT-SIGNATURE header
curl -s -X POST https://x402.ai.bio.xyz/api/deep-research/start \
  -H "Content-Type: application/json" \
  -H "PAYMENT-SIGNATURE: <base64-encoded x402 v2 payment payload>" \
  -d '{"message":"What is NAD+?","researchMode":"steering"}'
# → 200 with conversationId

# 4. Poll — requires SIWX auth (cannot use plain curl)
# The poll endpoint returns 401 with an SIWX challenge that must be signed
# by the paying wallet. Use the reference Python/Node scripts instead.
# See {baseDir}/references/siwx-protocol.md for the full protocol.

# 5. (Optional) Fetch feedback data for on-chain submission
curl -s https://x402.ai.bio.xyz/api/feedback/{conversationId}
```

## Polling with SIWX (TypeScript)

The poll endpoint requires SIWX authentication. Here is the TypeScript flow (from the reference implementation):

```typescript
import { createSiweMessage } from "viem/siwe";

type SiwxChallenge = {
  domain: string;
  uri: string;
  nonce: string;
  chainId: number;
  statement: string;
  version: "1";
  issuedAt: string;
  expirationTime: string;
};

async function signSiwxChallenge(challenge: SiwxChallenge, account: LocalAccount): Promise<string> {
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
  return Buffer.from(JSON.stringify({ message, signature })).toString("base64");
}

// Poll loop
let res = await fetch(`${BASE}/api/deep-research/${conversationId}`);

if (res.status === 401) {
  const { siwx } = await res.json();
  const siwxHeader = await signSiwxChallenge(siwx, account);
  res = await fetch(`${BASE}/api/deep-research/${conversationId}`, {
    headers: { "X-SIWX": siwxHeader },
  });
}

// For timeout retry (402 after SIWX), send both headers:
if (res.status === 402) {
  const paymentSig = /* sign fresh x402 payment */;
  // Fetch a fresh SIWX challenge
  const challengeRes = await fetch(`${BASE}/api/deep-research/${conversationId}`);
  const { siwx } = await challengeRes.json();
  const siwxHeader = await signSiwxChallenge(siwx, account);

  res = await fetch(`${BASE}/api/deep-research/${conversationId}`, {
    headers: { "X-SIWX": siwxHeader, "PAYMENT-SIGNATURE": paymentSig },
  });
}
```

## Reference implementation (x402)

A Python script for the x402 path is at `{baseDir}/references/python-script.md`. It delegates EIP-712 signing to a Node helper at `{baseDir}/references/research_signer.md`.

These scripts handle x402 payment negotiation and SIWX poll authentication. For the API key path, a simple `curl` or `httpx`/`requests` call with a Bearer header is all you need (see curl examples above).

Usage:

```bash
# Private key wallet
export WALLET_PRIVATE_KEY=0x...
python3 research.py "Your query" --mode steering

# CDP wallet
export CDP_API_KEY_ID=...
export CDP_API_KEY_SECRET=...
export CDP_WALLET_ADDRESS=0x...
python3 research.py "Your query" --mode smart

# Dry run (no payment)
python3 research.py --dry-run "test"

# Submit without polling
python3 research.py --mode steering --no-poll "test"
```
